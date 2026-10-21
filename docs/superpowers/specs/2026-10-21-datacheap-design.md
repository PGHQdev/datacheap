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

SvelteKit's origin check rejects cross-origin form-type POST bodies, including `text/plain`, unless the origin is in `csrf.trustedOrigins`. Customer origins change per site, so a static list does not fit, and trusting `*` removes the framework check from dashboard forms. Therefore the collector accepts only `application/json` through CORS with a cached preflight, and the framework origin check stays enabled. Unload delivery uses `fetch` with `keepalive`; validate it in target browsers and record measured loss. Confirm this behavior against the pinned SvelteKit version.

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

Simple funnels accept two to five page-path or event-name steps. Count a session once when it completes steps in order with at most thirty minutes from the first step to the last step; equal timestamps use a stable event-ID ordering. Show the definition next to the result. No cross-device funnel claim.

An owner can rotate site API keys, configure retention, configure one or more goal event names per site, export bounded CSV data, and delete a website with its data. Deletion first sets the site state to `deleting` and revokes its API keys and Stripe connection in one atomic write. Every write path, including the collector, the revenue API, webhook receipts, resolve tasks, and export jobs, writes only when the site is active, checked in the same statement, batch, or transaction as the write. The only exception is the bounded deletion job, which then removes the site's rows, because one request cannot delete a large site within D1 and serverless limits. No email infrastructure is needed for initial setup. Use Better Auth with its Drizzle SQLite/PostgreSQL adapters for owner credentials and sessions. A deployment-provided, high-entropy setup token gates first-owner creation; the database atomically permits only one bootstrap. Public signup is then disabled. Recovery is an operator CLI procedure using database access, not a public bypass endpoint. Runtime compatibility is an early validation requirement.

UI direction: restrained typography, accessible contrast, a consistent accent, compact metrics, clear tables, and useful empty states. Revenue and its attribution are prominent. No copied DataFast screen or branding.

## 4. Collection and metric contract

Browser collection uses a public site identifier. The collector accepts POST bodies up to 32 KiB and batches up to 20 events. It validates event IDs, names, timestamps, origins, string lengths, and a bounded primitive property map. Each request carries a payload schema version. An unknown version rejects the whole request, so a cached tracker of another version fails visibly. Within a known version, an unknown field rejects only its event. The response reports one result per event: accepted, duplicate, capped, or rejected with a code. An invalid event does not reject valid events in the same batch, and duplicates inside one batch count once. SQL remains parameterized.

Default privacy mode keeps a random identifier in sessionStorage, scoped to one browser tab. It uses no cookie, localStorage, fingerprint, or raw-IP persistence. The identifier survives reloads and same-tab navigation, including a same-tab return from a hosted checkout. A new tab or a later visit usually creates a new visitor, so unique counts are approximate and must be labeled. A tab opened with an opener starts with a copy of the opener's sessionStorage, and browsers may also copy it into a duplicated tab, so such tabs can share an identifier. Browser policy can block storage access; then the tracker uses an in-memory identifier for that page. The tracker catches its own errors, including storage and cookie failures, and never throws into the host page. In this mode, attribution covers only touches in the same tab. Optional persistent identification is enabled only after the site's consent integration calls the tracker consent API. Consent promotes the current tab identifier to a random first-party cookie, so consent does not create a second visitor in that tab. When the cookie exists, a new tab uses it, and other open tabs switch to it at their next event; their earlier events keep the old identifier. The cookie has a 90-day expiry. Revocation deletes the cookie, and the tracker starts a new tab identifier. Browsers can shorten cookies that scripts write; Safari ITP caps them at seven days, which limits multi-visit attribution for those visitors. Both modes explicitly document attribution limitations. No blanket legal-compliance claim.

Session IDs expire after thirty minutes of inactivity. Pageviews include initial load and SPA navigation, with duplicate navigation suppression. Events are uniquely keyed by site and event ID. The initial collector accepts timestamps up to 24 hours old and five minutes ahead; invalid timestamps fail validation. Event occurrence time controls reporting, while receipt time is retained for operations.

Public collection requires an allowlisted browser Origin. Server-originated events use a scoped secret API key. Origin can be spoofed outside browsers and is not proof of authenticity. Apply bounded database-backed rate counters keyed by site and a keyed, short-window hash of the trusted client address; counters store no raw address. Where a profile has no trusted client address, counters use the site key only. Host-level protection, where configured, runs before the database counter. A per-site daily accepted-event cap, set by the owner, protects database size; events over the cap get the `capped` result, and the dashboard shows dropped counts. Return 429 and Retry-After when limited. Do not return success before durable insertion. Storage outages return retryable 503 responses. Retry uses the same event IDs and bounded backoff.

Strip URL fragments, user info, and non-allowlisted query parameters. Preserve only approved campaign fields. Country comes from trusted host metadata when available. Where a host gives no trusted country metadata, a profile can use a bundled, freely licensed IP-to-country database or an operator-declared trusted proxy header. Without either, country is `unknown` and the dashboard states the reason. Confirm per host which metadata exists before the profile claims country support. Ignore forged provider headers on generic container targets. Device classification uses a small shared user-agent classifier. Basic bot exclusion is heuristic and documented.

Reporting uses UTC `[start, end)` intervals, a maximum 90-day query range, and stable ordering. The UI converts selected inclusive calendar days to that range and labels days as UTC. A site time zone is later work; SQLite has no time zone database, so it needs application-side offset handling that both database families verify. Pageviews count accepted pageview events; visitors count distinct identifiers in the range; sessions count distinct session IDs with activity in the range; conversions count sessions with at least one of the site's goal events, and each goal also has its own count. Revenue per visitor is net revenue in the selected currency divided by all visitors, returning null when there are no visitors. When the range contains revenue in other currencies, the UI labels the value as single-currency.

## 5. Revenue and attribution

Authenticated server revenue ingestion accepts a provider transaction ID, integer minor-unit amount, ISO currency, occurrence time, and an optional opaque visitor link. Never accept revenue claims through the public tracker. API keys have explicit site and action scope and are stored as hashes.

Persist both first-touch and last non-direct touch for the 30 days preceding a payment. First touch is the earliest eligible touch; last non-direct is the latest eligible campaign or external-referrer touch, falling back to direct only when no non-direct touch exists. Touches exclude referrers from the site's own allowed origins and from configured payment-provider hosts such as `checkout.stripe.com`, so a return from checkout never becomes a source. Freeze the selected attribution on the payment. Show unmatched payments as unattributed and report linkage coverage.

Use `(site, provider, transaction ID)` as the payment identity. A later submission with the same identity, amount, and currency does not create a payment and returns the stored payment with its current attribution; a different amount or currency returns 409. The stored occurrence time does not change. If the stored payment is unattributed and the later submission carries a visitor link, the link is applied once, attribution is computed and frozen, and the response shows that attribution. A link on an attributed payment is ignored. Store refunds as separate adjustments referencing the original payment, with their own unique provider IDs. A refund is effective while its latest status is `pending`, `requires_action`, or `succeeded`, and is not effective when `failed` or `canceled`. Stripe can move a refund from `succeeded` back to `requires_action` and then to `failed`. When a refund stops being effective, the ledger writes a reversing adjustment dated at the provider event time, so totals equal the sum of effective refunds. Status updates are deduplicated by provider event ID. An update older than the stored status time is ignored; for equal times, a final status wins, then the later provider event ID. `failed` and `canceled` are final, and nothing replaces them. A refund object fetched through the API gives its current status and replaces an older non-final status. The sum of effective refunds for a payment must not exceed its amount. The API returns 409 for an excess refund; a webhook excess refund is recorded as rejected and shown. Refunds inherit the original attribution and appear on their occurrence date. Do not silently combine currencies or infer exchange rates.

The first native payment integration is Stripe. Each site connects Stripe with its own webhook endpoint, signing secret, and restricted read-only Stripe key. Setup rejects a connection without the key, because subscription payments and early refunds need API lookups. The key is stored encrypted with a deployment secret and is never logged. If that secret is missing, readiness fails and Stripe ingestion stays closed. Pin the Stripe API version and confirm the lookups below against it. Verify signatures over the raw body before parsing or writing. The endpoint then stores a minimal receipt (event ID, event type, object IDs, receipt time) with a durable resolve task, and acknowledges the event. Receipts keep no payment payload and expire seven days after their task ends. The resolve task runs inline when possible and otherwise in the maintenance job; it retries with bounded backoff for up to seven days, then marks the task failed and shows it.

Normalize successful PaymentIntents as payments; do not also count Checkout Session or invoice notifications as new payments. In Checkout `payment` mode, the site key and visitor link go into `payment_intent_data.metadata`; that parameter exists only in `payment` mode. In `subscription` mode, they go into `subscription_data.metadata`. Subscription metadata does not reach the PaymentIntent, and from API version `2025-03-31.basil` the PaymentIntent and the Invoice no longer point to each other. Therefore the resolver looks up the Invoice Payment for the PaymentIntent, then the invoice and its subscription, and reads the site key and visitor link from the subscription metadata. A refund resolves through its payment: the stored payment when present, otherwise the payment fetched through the API. If that payment occurred before the site connected Stripe, the orphan rule below applies and the payment is not recorded. Otherwise the task records the payment first. Ownership is resolved before an event is recorded or skipped. The task records an event when the resolved site key matches the endpoint's site, or when no site key exists and the owner marks the Stripe account as exclusive to the site. It skips and counts an event with another site's key, or with no key on a shared account.

Renewal payments inherit the frozen attribution of the first attributed payment of their subscription, because renewals occur outside the 30-day touch window. A refund whose payment occurred before the site connected Stripe becomes orphaned: it is shown and excluded from totals, because historical import is later work. Replayed webhooks must not inflate totals. API ingestion of Stripe payments must use the same canonical IDs. Disputes and chargebacks are later work; until then the dashboard states that net revenue excludes them.

Generic revenue ingestion supports other payment providers without claiming native integrations. Native Polar, Lemon Squeezy, Shopify, historical import, foreign-exchange conversion, and subscription accounting are later work.

## 6. Persistence, cost, and operations

Logical tables include owner/auth sessions, sites, API keys, events, visitor touches, payments, refunds, webhook receipts, resolve tasks, maintenance cursors, and rate counters. IDs are strings; timestamps are UTC epoch milliseconds; money uses integers constrained to the safe transport range. Minor units follow the ISO 4217 exponent of each currency; JPY has none. Provider normalizers convert provider units to ISO minor units. Stripe sends ISK and UGX amounts with two extra decimal places, so the Stripe normalizer divides those amounts by 100 and rejects values not divisible by 100. SQLite and PostgreSQL physical types may differ.

Start with indexed raw events. Index site/time, site/visitor/time, and site/event-name/time; benchmark the write overhead. Avoid mandatory queues, Redis, R2, or ClickHouse until measurements demonstrate a need. Acknowledged data must be durable even on a terminating serverless request.

Raw events and attribution history default to 90-day retention. Revenue ledgers stay until the owner deletes them or their site, because their rows are small and paged exports, each bounded in size, need the full ledger. Reports keep the 90-day query range; longer revenue reports are later work. Owner-selected retention may reduce event history or set a revenue limit; reporting clearly shows available history. Deletion of a site removes its analytics and integration data and cancels its resolve tasks. Retention runs in bounded, resumable batches with visible last-success and failure status. D1 has a documented per-database size limit, so this profile targets modest workloads; do not promise unlimited scale or free hosting.

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

The public repository is `https://github.com/PGHQdev/datacheap`, but it has no buildable application yet. Therefore no deploy link can be finalized yet. Template publication, provider credentials, and real hosted checks are external prerequisites. Local manifests alone do not satisfy deployment acceptance.

Every profile needs documented backup/restore, secret rotation, upgrade, rollback, preview isolation, and data export. PostgreSQL restore should be tested from a dump; D1 recovery should be tested through its supported export/restore facilities. Application rollback never assumes a destructive database downgrade is safe.

## 8. Agent access and public discoverability

Expose a versioned API with OpenAPI, cursor pagination, explicit time ranges, scoped keys, and structured error codes. CLI commands cover sites, overview, breakdowns, funnels, and exports with JSON output and nonzero exit codes. Read-only stdio MCP tools call the same API using an operator-provided key; no remote OAuth service is required for the first release.

Public pages include semantic HTML, titles/descriptions, canonical URLs, sitemap, robots directives, llms.txt, Markdown documentation, and links to API/auth documentation. Authenticated dashboards are noindex and authorization-protected. Do not expose visitor records for crawler accessibility. Measure Lighthouse SEO and accessibility on public pages, and document agent-readiness checks actually implemented. Commerce and unrelated discovery protocols are not goals.

## 9. Verification and release acceptance

Critical automated checks cover malformed input, payload versions, per-event batch results, rate limits and daily caps, auth and site isolation, durable ingestion, retries, deduplication, tab-scoped identity, storage-blocked fallback, tracker errors contained from the host page, consent promotion and revocation, referrer exclusion, attribution windows, late visitor links, subscription renewal attribution, webhook site routing, Stripe connection key requirement and missing-secret failure, receipt expiry, ownership resolution before skip, subscription invoice lookup, shared and exclusive Stripe accounts, resolve tasks and their retries, refunds that arrive before their payment, refund status changes in any order, refund caps over effective refunds, orphaned refunds, site deletion write barrier, currency separation, provider minor-unit conversion, funnel order, UTC day boundaries, country fallback, bounded queries, bounded site deletion, migrations, retention, and API error contracts. Run the same fixtures against local D1 emulation and real PostgreSQL; emulated D1 success does not replace a hosted Cloudflare smoke test.

Browser verification covers setup, login/logout, JSON collector delivery including `keepalive` unload delivery, first-event detection, filters, empty/error states, mobile layout, keyboard navigation, exports, and token rotation. Avoid tests that merely assert implementation details or static copy.

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
- [SvelteKit configuration](https://svelte.dev/docs/kit/configuration): `csrf.checkOrigin` is deprecated for removal in 3.0; `csrf.trustedOrigins` governs cross-origin form-type POST bodies, including `text/plain`. This informs the JSON-only collector.
- [Stripe Checkout Session creation](https://docs.stripe.com/api/checkout/sessions/create) and [Stripe currencies](https://docs.stripe.com/currencies): `payment_intent_data` applies only in `payment` mode; ISK and UGX use a two-decimal representation for backward compatibility. These inform visitor linkage and minor-unit conversion.
- [Stripe Basil invoice payments change](https://docs.stripe.com/changelog/basil/2025-03-31/add-support-for-multiple-partial-payments-on-invoices): removes `invoice` from PaymentIntent and Charge and `payment_intent` from Invoice; the Invoice Payment object links them. [Stripe refunds](https://docs.stripe.com/refunds): refund statuses include `requires_action`, `failed`, and `canceled`, and a refund can return to `requires_action` after `succeeded`. These inform subscription lookup and the refund ledger.
- [MDN sessionStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage): a page with an opener starts with a copy of the opener's sessionStorage, and blocked storage throws `SecurityError`. This informs the identity limits and fallback.
- [WebKit ITP 2.1 analysis](https://www.simoahava.com/analytics/itp-2-1-and-web-analytics/): script-written cookies are capped at seven days. This is a 2019 secondary source; confirm current Safari behavior before publishing identity documentation.
- [Better Auth Drizzle adapter](https://better-auth.com/docs/adapters/drizzle): SQLite and PostgreSQL adapter support informs the auth choice; test complete runtime compatibility early.
- [Cloudflare deploy buttons](https://developers.cloudflare.com/workers/platform/deploy-buttons/) and [D1 limits](https://developers.cloudflare.com/d1/platform/limits/): native provisioning/bindings and binding-based migrations; bounded storage makes retention essential.
- [Vercel deploy buttons](https://vercel.com/docs/deploy-button) and [Neon managed integration](https://neon.com/docs/guides/vercel-managed-integration): database integration is part of deployment, not an assumed pre-existing secret.
- [Render deploy buttons](https://render.com/docs/deploy-to-render): Blueprint-based app/database setup; avoid silently auto-upgrading downstream instances.
- [Railway template publishing](https://docs.railway.com/templates/publish-and-share): a working deploy link requires publishing the template.
- [Coolify Compose](https://coolify.io/docs/applications/builds/docker-compose) and [Fly Launch](https://docs.fly.io/launch): their setup workflows require explicit packaging and validation.
- [Agent readiness](https://isitagentready.com/): informs public content and API discoverability; no claim that every emerging protocol is appropriate for this application.
