# datacheap.dev: proposed product and architecture

Date: 2026-10-21

Status: proposed for user review; implementation has not started. This is a design specification, not an execution plan or a claim of delivered functionality.

## 1. Intent and boundaries

The user requested an open-source clean-room implementation of the product category represented by DataFast, called datacheap.dev, deployable to Vercel, Cloudflare, Render, Railway, Coolify, and Fly.io. Product logic must remain common while infrastructure is appropriate to each host, including D1 on Cloudflare and Neon on Vercel.

Working assumptions for review:

- The first audience is technical founders and small SaaS operators who want to own their analytics and understand which acquisition sources produce revenue.
- Each deployment serves one owner with multiple websites. Shared hosted SaaS tenancy and team invitations come later.
- Apache-2.0 is the proposed license, favoring adoption and self-hosting. It is not yet applied.
- A useful first release delivers traffic-to-revenue attribution across all six hosting profiles. It does not duplicate every feature currently marketed by DataFast.
- Public behavior and documentation inform requirements. All implementation, copy, branding, and visual assets are original. This is an independent public-specification implementation, not a formally audited two-team clean-room process.

Success is a fresh installation that can receive an event, link a payment, show the correct source and amount, survive an upgrade, and export its data, with the same results on both database families.

## 2. Architecture alternatives

| Approach | Benefit | Cost | Decision |
| --- | --- | --- | --- |
| Shared core and UI, runtime and database adapters | Common behavior with native infrastructure | Two SQL dialects need contract verification | Recommended |
| Separate application per host | Maximum platform freedom | Six product implementations drift and multiply fixes | Reject |
| One container and PostgreSQL everywhere | Smallest infrastructure surface | Does not meet the Cloudflare D1 requirement | Reject |

Use pnpm workspaces, TypeScript, SvelteKit, Hono, and Drizzle. Hono exposes a portable Request/Response API mounted under SvelteKit server routes; it is not a separate mandatory service. SvelteKit owns pages and framework integration. Both call the same application services.

Proposed package boundaries:

| Location | Responsibility |
| --- | --- |
| `apps/web` | SvelteKit public pages, onboarding, dashboard, API mounting |
| `packages/core` | Domain models, validation, metric definitions, attribution, use cases, storage ports |
| `packages/api` | Hono handlers, auth policy enforcement, structured errors, OpenAPI |
| `packages/storage-d1` | D1 queries, SQLite schema, migrations |
| `packages/storage-postgres` | PostgreSQL queries and migrations; Neon and Node transports |
| `packages/tracker` | Browser collection, consent controls, attribution context |
| `packages/cli` | API client commands with JSON output |
| `packages/mcp` | Read-only stdio MCP wrapper over the authenticated API |
| `deploy` | Platform configurations, deployment/setup tools, smoke-check instructions |

The layout is proposed, not scaffolded. No platform SDK belongs in core. Runtime composition selects storage, secrets, metadata, and scheduling at build time. PostgreSQL-only dependencies must not enter the Cloudflare bundle. No dynamic import of arbitrary environment-provided module names.

Storage ports represent operations and guarantees, including `appendEvents`, `recordRevenue`, `recordRefund`, `queryOverview`, `queryBreakdown`, `queryFunnel`, `exportEvents`, and `pruneExpired`. They do not expose dialect-specific SQL to services. Multi-write operations commit atomically using D1 batches or PostgreSQL transactions. Deduplication is enforced by database constraints, not a check-then-insert race.

## 3. First-release experience

Onboarding: authenticate the owner, add a website and exact allowed origins, copy the script, observe a verified first event, connect revenue, inspect the first attributed payment. Show honest waiting/error states and a separately labeled sample-data preview. Production reports must never silently substitute samples.

The dashboard has a site selector, UTC date range, currency selector, active filters, and refresh status. Initial metrics are visitors, sessions, pageviews, conversions, net revenue, and revenue per visitor. A time series sits above sortable source, campaign, page, device, and country breakdowns. Unknown dimensions remain visible.

Simple funnels accept two to five page-path or event-name steps. Count a session once when it completes steps in order within a thirty-minute window; equal timestamps use a stable event-ID ordering. Show the definition next to the result. No cross-device funnel claim.

An owner can rotate site API keys, configure retention, export bounded data, and delete a website with its data. No email infrastructure is needed for initial setup. Use Better Auth with its Drizzle SQLite/PostgreSQL adapters for owner credentials and sessions. A deployment-provided, high-entropy setup token gates first-owner creation; the database atomically permits only one bootstrap. Public signup is then disabled. Recovery is an operator CLI procedure using database access, not a public bypass endpoint. Runtime compatibility is an early validation requirement.

UI direction: restrained typography, accessible contrast, a consistent accent, compact metrics, clear tables, and useful empty states. Revenue and its attribution are prominent. No copied DataFast screen or branding.

## 4. Collection and metric contract

Browser collection uses a public site identifier. The collector accepts POST bodies up to 32 KiB and batches up to 20 events. It validates event IDs, names, timestamps, origins, string lengths, and a bounded primitive property map. Unknown fields are rejected. SQL remains parameterized.

Default privacy mode uses an in-memory random identifier with no cookie, localStorage, fingerprint, or raw-IP persistence. It measures visits within a page/tab lifetime and must label unique counts as approximate; reloads can create new visitors. Optional persistent identification is enabled only after the site's consent integration calls the tracker consent API. It uses a random first-party identifier with a 90-day expiry and supports revocation. Both modes explicitly document attribution limitations. No blanket legal-compliance claim.

Session IDs expire after thirty minutes of inactivity. Pageviews include initial load and SPA navigation, with duplicate navigation suppression. Events are uniquely keyed by site and event ID. The initial collector accepts timestamps up to 24 hours old and five minutes ahead; invalid timestamps fail validation. Event occurrence time controls reporting, while receipt time is retained for operations.

Public collection requires an allowlisted browser Origin. Server-originated events use a scoped secret API key. Origin can be spoofed outside browsers and is not proof of authenticity. Apply bounded database-backed rate counters, with optional host-level protection; return 429 and Retry-After when limited. Do not return success before durable insertion. Storage outages return retryable 503 responses. Retry uses the same event IDs and bounded backoff.

Strip URL fragments, user info, and non-allowlisted query parameters. Preserve only approved campaign fields. Country comes from trusted host metadata when available; otherwise use `unknown`. Ignore forged provider headers on generic container targets. Device classification uses a small shared user-agent classifier. Basic bot exclusion is heuristic and documented.

Reporting uses UTC `[start, end)` intervals, a maximum 90-day query range, and stable ordering. The UI converts selected inclusive calendar days to that range. Pageviews count accepted pageview events; visitors count distinct identifiers in the range; sessions count distinct session IDs with activity in the range; conversions count sessions with an owner-selected goal event. Revenue per visitor is net revenue in the selected currency divided by visitors, returning null when there are no visitors.

## 5. Revenue and attribution

Authenticated server revenue ingestion accepts a provider transaction ID, integer minor-unit amount, ISO currency, occurrence time, and an optional opaque visitor link. Never accept revenue claims through the public tracker. API keys have explicit site and action scope and are stored as hashes.

Persist both first-touch and last non-direct touch for the 30 days preceding a payment. First touch is the earliest eligible touch; last non-direct is the latest eligible campaign or external-referrer touch, falling back to direct only when no non-direct touch exists. Freeze the selected attribution on the payment. Show unmatched payments as unattributed and report linkage coverage.

Use `(site, provider, transaction ID)` as the payment identity. A duplicate with the same payload returns the existing result; a conflicting amount or currency returns 409. Store refunds as separate adjustments referencing the original payment, with their own unique provider IDs. Refunds inherit the original attribution and appear on their occurrence date. Do not silently combine currencies or infer exchange rates.

The first native payment integration is Stripe. Verify signatures over the raw body before parsing or writing. Normalize successful PaymentIntents as payments; do not also count Checkout Session or invoice notifications as new payments. Carry datacheap visitor metadata into the PaymentIntent through documented checkout configuration. Normalize successful refunds by refund ID. Out-of-order refunds remain pending until their payment arrives; durable pending state is observable and retriable. Replayed webhooks must not inflate totals. API ingestion of Stripe payments must use the same canonical IDs.

Generic revenue ingestion supports other payment providers without claiming native integrations. Native Polar, Lemon Squeezy, Shopify, historical import, foreign-exchange conversion, and subscription accounting are later work.

## 6. Persistence, cost, and operations

Logical tables cover owner/auth sessions, sites, API keys, events, visitor touches, payments, refunds, webhook receipts, pending adjustments, maintenance cursors, and rate counters. IDs are strings; timestamps are UTC epoch milliseconds; money uses integers constrained to the safe transport range. SQLite and PostgreSQL physical types may differ.

Start with indexed raw events. Index site/time, site/visitor/time, and site/event-name/time; benchmark the write overhead. Avoid mandatory queues, Redis, R2, or ClickHouse until measurements demonstrate a need. Acknowledged data must be durable even on a terminating serverless request.

Raw events and attribution history default to 90-day retention. The first release keeps revenue ledgers for 365 days. Owner-selected retention may reduce either period; reporting clearly shows available history. Deletion of a site removes its analytics and integration data. Retention runs in bounded, resumable batches with visible last-success and failure status. D1 has a documented per-database size limit, so this profile targets modest workloads; do not promise unlimited scale or free hosting.

The initial benchmark target is 100,000 events per site per month and a 90-day overview response below two seconds at p95 on a documented paid baseline. These are validation targets, not measured capabilities or a promise about free tiers. Record storage size, query plan, rows scanned, write amplification, and host cost assumptions. If D1 misses the target, reduce scans or add verified rollups before making scale claims.

Health checks distinguish process liveness and database/schema readiness without exposing secrets. Logs contain request IDs and error categories, not customer payloads. Schema changes are versioned and additive where possible. No migration runs on every request. Startup blocks incompatible schemas; deployment failures do not quietly serve partial functionality.

## 7. Deployment profiles

| Target | Runtime/database | Provisioning and migration | Maintenance |
| --- | --- | --- | --- |
| Cloudflare | SvelteKit Worker + static assets; D1 | Wrangler binding; Deploy to Cloudflare; migrations reference the binding name | Scheduled Worker handler invokes shared retention job |
| Vercel | SvelteKit Functions + CDN; Neon PostgreSQL | Deploy button requests Neon product; connection secrets injected; serialized, retryable migration step against the target branch | Authenticated Vercel Cron endpoint |
| Render | Node Docker image; Render PostgreSQL | Blueprint provisions app/database; pre-deploy migration command | Dedicated scheduled job using the same image |
| Railway | Node Docker image; PostgreSQL service | Published template wires database variables; pre-deploy migration command | Scheduled maintenance service |
| Coolify | Node Docker image; PostgreSQL container | Compose application, persistent database volume, generated secrets, one-shot migration service | Maintenance container runs bounded jobs on a timer |
| Fly.io | Node Docker image on Machines; Fly Managed Postgres | Launch/setup automation creates or attaches database; release command migrates | Dedicated maintenance process with database lease |

All timers use a database-backed lease so concurrent triggers do not overlap. Neither correctness nor retention depends on a process-local lock. Scheduler failures are visible in the dashboard and logs. Public collection remains accessible even if dashboard routes require login.

One-click acceptance means that after normal platform authentication, plan selection, and secrets entry, the flow provisions or attaches the database, applies migrations, deploys a reachable app, and opens protected onboarding without manual source edits. Installing the script and connecting Stripe are separate product onboarding steps.

Cloudflare, Vercel, and Render have documented button flows. Railway needs an actual published template ID. Coolify needs an instance-specific import flow. Fly's documented baseline is CLI launch; an automated GitHub workflow or provisioning tool needs validation before it can be called one-click. All six remain required targets, but unsupported shortcuts must never be represented by decorative buttons.

The empty workspace has no Git remote or public repository URL. Therefore no deploy link can be finalized yet. Template publication, provider credentials, and real hosted checks are external prerequisites. Local manifests alone do not satisfy deployment acceptance.

Every profile needs documented backup/restore, secret rotation, upgrade, rollback, preview isolation, and data export. PostgreSQL restore should be tested from a dump; D1 recovery should be tested through its supported export/restore facilities. Application rollback never assumes a destructive database downgrade is safe.

## 8. Agent access and public discoverability

Expose a versioned API with OpenAPI, cursor pagination, explicit time ranges, scoped keys, and structured error codes. CLI commands cover sites, overview, breakdowns, funnels, and exports with JSON output and nonzero exit codes. Read-only stdio MCP tools call the same API using an operator-provided key; no remote OAuth service is required for the first release.

Public pages include semantic HTML, titles/descriptions, canonical URLs, sitemap, robots directives, llms.txt, Markdown documentation, and links to API/auth documentation. Authenticated dashboards are noindex and authorization-protected. Do not expose visitor records for crawler accessibility. Measure Lighthouse SEO and accessibility on public pages, and document agent-readiness checks actually implemented. Commerce and unrelated discovery protocols are not goals.

## 9. Verification and release acceptance

Critical automated checks cover malformed input, auth and site isolation, durable ingestion, retries, deduplication, attribution windows, refunds, currency separation, funnel order, bounded queries, migrations, retention, and API error contracts. Run the same fixtures against local D1 emulation and real PostgreSQL; emulated D1 success does not replace a hosted Cloudflare smoke test.

Browser verification covers setup, login/logout, first-event detection, filters, empty/error states, mobile layout, keyboard navigation, exports, and token rotation. Avoid tests that merely assert implementation details or static copy.

A first release is complete only when:

1. Original source and the agreed open-source license are published.
2. Fresh D1 and PostgreSQL installations produce equivalent analytics for shared fixtures.
3. A real tracker event and a signed test payment produce the expected attributed revenue; retries and refunds preserve correct totals.
4. Unauthorized API access, cross-site access, invalid signatures, and duplicate bootstrap attempts are rejected.
5. Builds pass for Cloudflare, Vercel, and the common Node container.
6. Each of the six deployment flows is exercised in a real account and the exact setup steps recorded.
7. Backup, restore, retention, upgrade, health checks, and scheduler failures are verified.
8. CLI/MCP access and documented public discovery endpoints work without weakening dashboard authorization.

Order of architectural milestones: shared core and D1/PostgreSQL contract; collection and owner onboarding; reporting and revenue; deployment profiles and agent clients; hosted acceptance. A detailed implementation plan follows design approval.

## 10. Research and decisions

Sources reviewed on 2026-10-21. Public feature claims change over time.

- [DataFast public product page](https://datafa.st/): revenue attribution, events, funnels, and agent access informed the category scope. This is a short feature reference, not a copied specification. Our release boundaries and architecture are our own decisions.
- [Lenny: determining an activation metric](https://www.lennysnewsletter.com/p/how-to-determine-your-activation-metric): distinguish a plausible milestone from a demonstrated retention predictor. Our hypothesis is that a first attributed payment is useful activation; validate it with consenting pilot users rather than installing mandatory telemetry in self-hosted instances.
- [READ Make](https://readmake.com): licensed guidance was accessible and informed a small, usable first release and low operational complexity. The licensed text is not stored in this public repository. Optional managed hosting and paid deployment/support are future monetization hypotheses; no evidence of willingness to pay has been collected.
- [Hotjar screen via Mobbin](https://mobbin.com/screens/b368f8ba-7b2f-4065-bc3c-9d6702e5b992): inspected date/filter controls, compact metrics, and ranked breakdowns. [Wix screen via Mobbin](https://mobbin.com/screens/ddb749b0-c243-4272-91bb-1e0ceb702123): inspected separation of summary metrics and traffic-source details. Apply these general usability patterns through original design.
- [Svelte adapters](https://svelte.dev/packages) and [SvelteKit migration documentation](https://svelte.dev/docs/kit/migrating-to-sveltekit-3): platform adapters exist, but version compatibility and runtime bindings must be checked when pinning dependencies. Do not mix older integration snippets with a newer framework release.
- [Better Auth Drizzle adapter](https://better-auth.com/docs/adapters/drizzle): SQLite and PostgreSQL adapter support informs the auth choice; test complete runtime compatibility early.
- [Cloudflare deploy buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/) and [D1 limits](https://developers.cloudflare.com/d1/platform/limits/): native provisioning/bindings and binding-based migrations; bounded storage makes retention essential.
- [Vercel deploy buttons](https://vercel.com/docs/deploy-button) and [Neon managed integration](https://neon.com/docs/guides/vercel-managed-integration): database integration is part of deployment, not an assumed pre-existing secret.
- [Render deploy buttons](https://render.com/docs/deploy-to-render): Blueprint-based app/database setup; avoid silently auto-upgrading downstream instances.
- [Railway template publishing](https://docs.railway.com/templates/publish-and-share): a working deploy link requires publishing the template.
- [Coolify Compose](https://coolify.io/docs/applications/builds/docker-compose) and [Fly Launch](https://docs.fly.io/launch): their setup workflows require explicit packaging and validation.
- [Agent readiness](https://isitagentready.com/): informs public content and API discoverability; no claim that every emerging protocol is appropriate for this application.
