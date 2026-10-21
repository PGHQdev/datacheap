# datacheap.dev

Open-source web and revenue analytics, designed to run in your own infrastructure.

**Status: project initialization and proposed design. There is no runnable application or verified deployment template yet.**

The intended product connects traffic sources, campaigns, events, and conversions to revenue. It is an independent implementation informed by public analytics-product behavior, with original code and design.

## Shared product, native infrastructure

One TypeScript domain core, one SvelteKit interface, and a portable Hono API. Hosting differences belong in deployment profiles and storage adapters.

| Target | Proposed runtime | Proposed database | Deployment path to implement |
| --- | --- | --- | --- |
| Cloudflare | Workers + static assets | D1 | Deploy to Cloudflare button with bindings and migrations |
| Vercel | Functions + CDN | Neon PostgreSQL | Deploy button with Neon integration |
| Render | Node container | Render PostgreSQL | Blueprint containing app and database |
| Railway | Node container | Railway PostgreSQL service | Published app + database template |
| Coolify | Node container | PostgreSQL container with persistent volume | Importable Compose application |
| Fly.io | Node container on Machines | Fly Managed Postgres | Launch configuration and automated setup workflow |

These are proposed targets, not claims of deployed support. Fly and Coolify need their own setup flows; a link to generic platform documentation is not a one-click deployment. Hosting accounts and infrastructure charges remain the operator's responsibility.

## Proposed first release

- Private, multi-site dashboard for one deployment owner.
- Lightweight pageview and custom-event tracking with campaign attribution.
- Visitor/session metrics with explicit identity and consent behavior.
- Revenue API, Stripe webhook ingestion, refunds, and source attribution.
- Simple ordered funnels, date filters, source/page/campaign breakdowns, CSV export.
- Scoped API tokens, OpenAPI documentation, a CLI, and read-only MCP tools.
- Deployment profiles, migrations, retention, health checks, and recovery documentation for all six targets.

Full DataFast feature parity is not claimed. Native mobile SDKs, session replay, social listening, additional payment integrations, and hosted subscription billing are outside the proposed initial release.

## Project documents

- [Agent working instructions](AGENTS.md)
- [Proposed architecture, scope, and acceptance criteria](docs/superpowers/specs/2026-10-21-datacheap-design.md)

The proposed open-source license is Apache-2.0, pending design review. No release or hosted service has been published.
