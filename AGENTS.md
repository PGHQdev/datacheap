# datacheap.dev

## Mission and current state

Build an open-source, independently implemented web and revenue analytics product. Keep product behavior common across Cloudflare, Vercel, Render, Railway, Coolify, and Fly.io; specialize infrastructure behind adapters.

This repository currently contains project instructions and a proposed architecture, not a runnable application. Read `README.md` and `docs/superpowers/specs/2026-10-21-datacheap-design.md` before implementation. Do not describe proposed features or deployment paths as shipped.

## Working agreements

- Use pnpm; Bun is acceptable for tooling. Do not introduce npm or Yarn lockfiles.
- Prefer SvelteKit for the application, Hono for the portable HTTP API, and TypeScript for the shared core. Cloudflare is the reference deployment. Containers and VPS hosting are first-class targets.
- Keep the scope of tests on critical behavior: authentication, authorization, ingestion validation, attribution, money, API responses, migrations, and database adapter contracts. Use browser checks for presentation changes rather than superficial component tests.
- Verify relevant builds and behavior before claiming completion. Distinguish local checks, emulated platform checks, and real hosted smoke tests.
- Never commit secrets, production analytics, customer information, or licensed research content.
- Use relevant Lenny's Data, READ Make, and Mobbin research proportionally. Inspect design images and cite sources for significant decisions. Record unavailable access honestly. Domain checks never authorize purchases.

## Clean-room boundaries

- Use public product behavior and documentation to define requirements. Write original code, wording, styles, icons, and assets.
- Do not copy DataFast source, bundles, private endpoints, branding, screenshots, or proprietary assets into the product.
- Record public references and distinguish observed behavior from our own decisions. Do not imply affiliation or complete feature parity.
- Keep the public product name `datacheap.dev`; package names may use `@datacheap/*` when published and available.
- Proposed license: Apache-2.0. Confirm it as part of design review before adding the license and publishing code.

## Architectural rules

- Shared core owns event normalization, metric definitions, attribution, money rules, authorization policy, and application use cases.
- Core modules must not import platform SDKs, environment globals, filesystem APIs, database drivers, or SvelteKit modules. Inject time, IDs, storage, and platform context.
- The UI and API share use cases. Do not implement business rules separately in Svelte routes, HTTP handlers, and deployment entrypoints.
- Provider implementations own database connections, SQL dialects, migrations, trusted request metadata, scheduling, and deployment configuration.
- Cloudflare uses D1. Vercel uses Neon PostgreSQL. Render, Railway, Coolify, and Fly.io use PostgreSQL through their deployment profiles.
- PostgreSQL providers share query semantics and migrations; Neon may require a different transport. SQLite/D1 has separate migrations and query implementations with the same observable contract.
- No provider switches in domain logic. Use build-selected server composition modules so incompatible drivers cannot enter the wrong bundle.
- Prefer explicit operations such as `recordRevenue` and `queryOverview` over a generic lowest-common-denominator database wrapper.
- Avoid optional infrastructure becoming mandatory: no Redis, queue, object store, paid geolocation service, or external identity provider required for the initial release.

## Data correctness and access

- All site data access must enforce site ownership or explicit token scope. Public collection credentials never grant dashboard, identity, or revenue-write access.
- Validate and bound events before storage. Treat Origin checks as browser abuse mitigation, not authentication.
- Persist ingestion before acknowledging success. Retries must not double-count events, payments, or refunds.
- Store monetary amounts as integer minor units plus currency; never silently sum different currencies or use floating point for ledger math.
- Preserve unattributed revenue explicitly. Never invent a channel when visitor linkage is missing.
- Strip URL fragments, credentials, and non-allowlisted query parameters. Do not log raw request bodies, auth tokens, IP addresses, or payment payloads.
- Dashboard sessions require secure cookies, CSRF protections, revocation, and login rate limits. Use an established authentication library.
- Bound query ranges, pagination, ingestion batches, retention jobs, and export sizes. Use SQL parameters; never interpolate user input into SQL.
- Fail closed on missing secrets, invalid adapter configuration, or incompatible schema versions. Never fall back to demo data or an in-memory database in production.

## Deployability and operations

- A deployment profile includes runtime, database provisioning or attachment, secrets, migrations, health checks, maintenance, upgrades, and recovery instructions.
- Maintain one feature contract across targets. Performance differences must be measured and documented.
- Never invent a GitHub URL, Railway template ID, integration ID, or working deployment badge. Publish links only after their prerequisites exist.
- Call a route one-click only after exercising its platform setup flow. Explicitly list account, billing, secret, domain, and provider-integration steps.
- Migrations run as an explicit deployment step and must tolerate retries. Preview deployments must never migrate production databases.
- Report scheduler failures and retention lag. Document backup/restore for every supported database profile.

## Agent readiness and public pages

- Provide semantic accessible HTML, keyboard-operable controls, machine-readable API documentation, scoped API tokens, and structured errors.
- Provide public docs, robots.txt, sitemap.xml, llms.txt, and Markdown documentation. Keep authenticated analytics private regardless of crawler permissions.
- Add CLI and MCP clients over the same API. Do not advertise OAuth, commerce, or other discovery capabilities before implementation.
- Measure public-page Lighthouse SEO and accessibility using pnpm dlx; report the actual environment and scores. Never fabricate metrics.

## Commands

No package manifest or executable application exists yet. Add the actual install, development, build, migration, and verification commands here when they work; do not present proposed commands as available.
