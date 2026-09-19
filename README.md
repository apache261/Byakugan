# Byakugan

Lightweight Go REST microservice for ISO 20022-style bank transfer fraud checks
and a separate AML/CFT monitoring subsystem.

## Moved as private Repo :( 
For demo contact me: lynolibarra@gmail.com

## About Byakugan

Byakugan is a bank fraud detection microservice created during my free time. The
name is inspired by the Byakugan ability from the Naruto anime, associated with
the Hyuga clan characters such as Hinata and Neji, because the software is meant
to help admins "see" transfer risk, account behavior, rule hits, and decision
signals clearly.

Byakugan also supports jurisdiction-neutral anti-money-laundering and
counter-terrorist-financing (AML/CFT) monitoring. Its AML ledger, scenarios,
alerts, investigations, and reports are separate from synchronous fraud
decisions: AML findings do not authorize, reject, or hold payments.

The project is also heavily inspired by Tazama and by the reality that there are
only a limited number of open-source bank fraud detection applications available
for learning, prototyping, and lightweight deployments.

I developed this during my free time (since 2024).

## Features

- Canonical JSON intake for `pacs.008`-style account transfers
- Single deployable microservice: `byakugan-api`
- Synchronous `allow`, `review`, or `block` decisions
- Account allow/block controls and per-account config
- Portable JSON rule DSL for fraud/validation checks
- Explainable velocity, beneficiary-history, route, remittance, execution-date,
  and risk-tier rule fields
- Current-inclusive 24-hour incoming and outgoing PHP account accumulation
  rules that review totals above 50,000
- Rule simulation endpoint for testing draft rules before saving
- Dashboard-ready REST endpoints
- PostgreSQL schema migrations
- Dedicated AML canonical domain, append-only persistence, ingestion,
  versioned scenarios/screening, durable monitoring, and explainable alerts
- AML investigation cases, evidence, four-eyes report approval, legal holds,
  retention controls, and historical imports with authorized backfills
- Role-restricted AML/CFT workflows in the standalone admin dashboard
- Optional Redis client wiring for optimized deployments

## Stack Used

- Language: Go 1.18
- API style: REST over Go `net/http`
- Data model: ISO 20022-style canonical JSON for `pacs.008` transfers
- Primary database: PostgreSQL schema and migration SQL
- Cache/optimization: Redis-ready cache and idempotency wiring
- Admin UI: static HTML/JavaScript with a full W2UI shell, jQuery UI modal forms, manual CSS, and Chart.js
- API docs: OpenAPI 3.0 YAML with Swagger UI
- Runtime/deployment: Docker and Docker Compose
- Testing: Go unit/API tests plus lightweight JavaScript syntax checks

## Run

```sh
go run ./cmd/byakugan-api
```

The service defaults to an in-memory store for local development. Set
`DATABASE_URL` to enable PostgreSQL persistence and database readiness checks.
Set `REDIS_ADDR` to enable Redis-oriented cache readiness wiring.

Environment variables are listed in `.env.example`. For local shell usage,
export only the values you need; Docker Compose sets its own defaults.

## Docker Compose

```sh
docker compose up --build
```

The root `VERSION` file is the main product version. To build the API and
dashboard with that version and one automatically generated UTC build number,
use:

```sh
sh scripts/build-images.sh
```

CI may set `BUILD_NUMBER` to an immutable pipeline run number. The optional
`API_BUILD_NUMBER` and `DASHBOARD_BUILD_NUMBER` values override one component.
Direct Compose builds also generate build numbers when those values are empty,
but the wrapper ensures both images receive the same build number.

The API reports its version and build number from `/healthz` and `/readyz`.
The dashboard displays its own version and build number in the W2UI shell.

- Byakugan API service: `http://localhost:9095`
- Administrative dashboard: `http://localhost:9096`
- PostgreSQL: `localhost:3309`
- Redis: `localhost:6379`

The dashboard port is bound to localhost by default. Expose it only through the
approved internal administrative network and production HTTPS termination.
Port `9095` is the Go API and operational endpoint, not a UI entry point.

The PostgreSQL Docker image includes all `migrations/*.up.sql` files and runs
them through `/docker-entrypoint-initdb.d/` when the `postgres_data` volume is
first created. This creates the schema and installs the default Philippines/PHP
fraud rules.
Docker Compose sets `DATABASE_URL`, so the API uses PostgreSQL persistence in
the composed environment.

If you change migrations after the database volume already exists, recreate the
volume:

```sh
docker compose down -v
docker compose up --build
```

Current operations, licensing, dashboard permissions, and the AML investigation
reporting, historical-import, and operational-index foundation require schema version 34. For environments with
retained data, apply the ordered migration files through the deployment
migration job instead of recreating the volume. `/readyz` remains unavailable
when the required schema, seeded rules, or required configurations are missing.

## OpenAPI

The OpenAPI 3.0 spec with request and response samples is in:

```sh
docs/openapi.yaml
```

When the API is running, Swagger UI is available at:

```sh
http://localhost:9095/docs
```

The raw OpenAPI YAML is served from:

```sh
http://localhost:9095/docs/openapi.yaml
```

## AML/CFT APIs

**AML support status:** implementation Phases 0–8 are complete. Phase 9 security,
scale, and release readiness is in progress; its production-like staging,
restore, performance, and operator approval evidence remains outstanding. See
[AML_plan_phase.md](AML_plan_phase.md) and the
[AML release checklist](docs/aml/release-checklist.md). The implemented AML core
is jurisdiction-neutral. Institution-specific sanctions sources, reporting
formats, thresholds, and submission connectors require separate configuration
or adapters; their presence is not implied by the built-in workflows.

Internal-only AML APIs cover customer risk snapshots, accounts, an append-only
transaction ledger, scenario and screening management, asynchronous monitoring,
alerts, confidential investigations, reports, legal holds, retention, and
imports. They are separate from fraud transfer checks:

```text
POST /v1/aml/customers/{id}/snapshots
GET  /v1/aml/customers/{id}
GET  /v1/aml/customers/{id}/snapshots
PUT  /v1/aml/accounts/{id}
GET  /v1/aml/accounts/{id}
POST /v1/aml/transactions
POST /v1/aml/transactions/batch
POST /v1/aml/transactions/iso20022?source_system=payments-gateway
GET  /v1/aml/transactions
GET  /v1/aml/transactions/{id}
POST /v1/aml/scenarios
GET  /v1/aml/scenarios[/{id}]
POST/GET /v1/aml/scenarios/{id}/versions
POST /v1/aml/scenarios/{id}/activate|deactivate|simulate|dry-run
POST /v1/aml/policy-packs/import
GET  /v1/aml/policy-packs[/{id}/export]
POST/GET /v1/aml/screening-results
GET  /v1/aml/screening-results/{id}
PATCH /v1/aml/screening-results/{id}/disposition
GET  /v1/aml/monitoring/options|metrics
GET  /v1/aml/monitoring-runs[/{id}]
POST /v1/aml/monitoring-runs/reconcile
POST /v1/aml/monitoring-runs/{id}/replay|cancel
GET  /v1/aml/alerts[/{id}]
POST /v1/aml/alerts/{id}/assign|triage|investigate|escalate|close
POST/GET /v1/aml/cases
GET  /v1/aml/cases/{id}
POST /v1/aml/cases/{id}/alerts|assign|investigate|escalate|close|reopen
POST/GET /v1/aml/cases/{id}/notes|attachments
GET  /v1/aml/cases/{id}/attachments/{attachment_id}
GET  /v1/aml/cases/{id}/evidence
POST/GET /v1/aml/reports
GET  /v1/aml/reports/{id}
POST /v1/aml/reports/{id}/versions|review|approve|export|withdraw
POST/GET /v1/aml/legal-holds
GET  /v1/aml/legal-holds/{id}
POST /v1/aml/legal-holds/{id}/release
POST /v1/aml/retention/preview|purge
POST/GET /v1/aml/mapping-profiles
GET  /v1/aml/mapping-profiles/{id}
POST /v1/aml/mapping-profiles/{id}/versions
POST/GET /v1/aml/imports
GET  /v1/aml/imports/options
GET  /v1/aml/imports/{id}
POST /v1/aml/imports/{id}/validate|commit|retry|cancel|monitor
GET  /v1/aml/imports/{id}/errors
```

Writes require `aml:transactions:write`; reads require `aml:read`. Admin-session
writes require an AML analyst, supervisor, administrator, or super-admin role.
Every AML route also enforces the private/internal client boundary. Exact money
amounts are JSON strings, batches contain 1–500 records, and transaction lists
use opaque signed cursors. Set `AML_CURSOR_SIGNING_SECRET` to the same secret on
all production replicas so cursors survive restarts and cross-replica requests.

Scenario reads require `aml:read`; scenario and policy administration require
`aml:admin` and an AML administrator or super-admin session. Confidential
screening operations require `aml:investigate` and an AML analyst-capable
session. Version and disposition changes require `If-Match`. Policy-pack
imports are immutable, content-hashed, compatibility checked, and
non-activating. Simulation and dry-run return explainable observations only;
they cannot create alerts or change scenario state. Monitoring replay,
reconciliation, and cancellation require `aml:admin`; confidential alert reads
and workflow actions require `aml:investigate` and use `If-Match`.

AML cases, notes, protected attachments, and evidence exports require
`aml:investigate`. Reports can only originate from suspiciously closed AML
cases. Approval/export/withdrawal require `aml:approve` and an interactive AML
supervisor or super-admin session; the current version author can never approve
that version. Holds and retention require `aml:admin`, while purge additionally
requires an interactive AML admin session and a recent actor-bound preview
token. Import and mapping routes require `aml:admin` plus an AML administrator
or super-admin role. Phase 7 streams CSV, JSON arrays, NDJSON, and `pacs.008`
into canonical customers, snapshots, accounts/profiles, parties,
relationships, transactions, screening/watchlist results, and supported legacy
alerts/cases. Corrected quarantine retries retain attempt lineage and never
replay prior accepted records. Monitoring remains suppressed by default; an
eligible completed import needs a separate, bounded historical-backfill command
before scenarios can create deduplicated AML alerts.
Set `AML_BLOB_ROOT` to durable, access-restricted storage. Attachment and import
source bytes stay there while PostgreSQL stores metadata, hashes, checkpoints,
quarantine records, and reconciliation summaries. Files use mode `0600`
beneath a `0700` root. The Compose deployment runs a
one-shot `aml-storage-init` service before the API to assign the named volume to
the API's fixed non-root identity (`10001:10001`). This also repairs ownership
on volumes created by older images.

Scheduled monitoring defaults to 15-minute incremental runs and daily
reconciliation with an event-time overlap. Configure it with the
`AML_MONITORING_*` variables in `.env.example`. `AML_DEPLOYMENT_ID` and
`AML_CURSOR_SIGNING_SECRET` must be stable and shared by production replicas.
Optional real-time monitoring returns a `monitoring_reference` after durable
ledger ingestion; evaluation remains asynchronous. AML endpoints and workers
never return or alter a payment authorization decision, and the `pacs.008`
adapter does not invoke the fraud transfer-check endpoint.

In Docker Compose, protected API calls use the local key `dev-secret`. Swagger UI
automatically sends it for `/v1/*` requests.

## Dashboard

The plain HTML/CSS/JavaScript Byakugan administrative dashboard is available
from its independent Nginx container at:

```sh
http://localhost:9096
```

The dashboard container should be exposed only to approved private/internal
clients. Go independently enforces the private-client restriction on admin and
privileged API operations after validating the trusted proxy address.

Dashboard admins log in with username/password. New and updated database-backed
admin accounts store bcrypt password hashes. Existing legacy SHA-256 admin
hashes remain valid for compatibility until the password is changed. Defaults
are `admin` / `admin` for local development; set these in production:

```sh
ADMIN_USERNAME=admin
ADMIN_PASSWORD=change-this-password
ADMIN_SESSION_COOKIE_SECURE=true
```

Use `ADMIN_SESSION_COOKIE_SECURE=false` only for local HTTP development. The
standalone Nginx dashboard preserves the Go service's session cookie unchanged,
so production HTTPS deployments must enable the secure attribute in Go.

The dashboard proxy overwrites client-supplied forwarding headers and sends the
original client address to Go. Go accepts those headers only when the immediate
sender matches `TRUSTED_PROXY_CIDRS`. Leave that setting empty for direct API
deployments and use the narrowest proxy CIDRs possible in production. Docker
Compose assigns the dashboard proxy `10.253.0.10` on `10.253.0.0/24` and
trusts only `10.253.0.10/32`. If that subnet overlaps another Docker network,
set `BYAKUGAN_NETWORK_SUBNET`, `DASHBOARD_NETWORK_IPV4`, and
`TRUSTED_PROXY_CIDRS` together to a free subnet, an address in that subnet,
and the same address as a `/32`, respectively.

It uses jQuery, jQuery UI, W2UI, and Chart.js from approved public CDNs, with manual product CSS.
There is no React, Angular, or
npm-based frontend build.

A standalone dashboard is deployed from top-level `dashboard/`. Phases 1
through 11 provide the independent shell, Nginx proxy, trusted
session boundary, shared browser services, API coverage register, working
administrative modules for monitoring, cases, rules, accounts, activity,
integrations, watchlists, configuration, and AML/CFT, the visual rule builder,
development simulator, accessibility controls, and automated quality gates.
It runs in its own Nginx container, serves its own static assets, and
reverse-proxies `/admin/*` and `/v1/*` to Byakugan. This preserves the existing
same-origin `bfd_admin_session` authentication flow without adding CORS while
removing UI asset delivery and UI behavior from the Go application. The
Go binary no longer contains or serves dashboard, rule-builder, rule-help,
or simulator assets. See [ui-roadmap.md](ui-roadmap.md) for the completed
migration plan and [docs/operations.md](docs/operations.md) for deployment and
rollback procedures.

The Accounts module accepts CSV imports through **Import CSV**. Required
columns are `id,name`; optional columns are `iban,bban,status,risk_tier` and
`metadata_json`. Imports are limited to 4 MiB and 5,000 account rows and return
an independent result for each row. The Config action returns the effective
global defaults when an account has no saved override yet.

The AML/CFT module appears in the sidebar only for administrators assigned the
`aml` dashboard module. AML-specific roles receive that module by default;
other administrators can be granted it through Configuration > Administrator
Accounts. AML actions remain further restricted by the server-returned role
capabilities. Rebuild the independently deployed dashboard image after adding
or upgrading the AML module assets.

The standalone dashboard remains plain HTML, CSS, and JavaScript. It may use
jQuery UI, W2UI, Chart.js, and other focused external JavaScript
or CSS libraries where useful, but it will not use React, Vue, Angular,
Tailwind, a bundler, or a frontend build step.

Run the independent dashboard quality gates with:

```sh
sh dashboard/tests/run-static.sh
sh dashboard/tests/run-live.sh # after the operator starts Docker Compose
```

The static gate validates runtime configuration, accessibility contracts, the
ES-module graph, JavaScript syntax, Nginx configuration, image policy, dynamic
dropdown policy, and accidental credential inclusion. The live gate validates
same-origin authentication and proxy behavior plus the migrated operator
workflows. CI keeps Go/OpenAPI contract tests, dashboard static checks,
independent image health, and full-stack smoke checks as separate jobs.

Admins can approve reviewed transfer checks from the dashboard review queue. The
same action is also available through:

```sh
POST /v1/transfer-checks/{id}/approve
```

Reviewed and blocked transfer checks automatically create one fraud case per
transfer check. Cases are visible from the dashboard Cases view and through:

```sh
GET /v1/cases
GET /v1/cases?status=investigating&decision=review&priority=high&limit=25&offset=0
GET /v1/cases/{id}
PATCH /v1/cases/{id}
GET /v1/cases/{id}/notes
POST /v1/cases/{id}/notes
GET /v1/cases/{id}/attachments
POST /v1/cases/{id}/attachments
GET /v1/cases/{id}/attachments/{attachment_id}
```

Case listing supports pagination plus status, decision, priority, queue,
assignee, text search, and `include_closed=true` filters for resolved cases.
Case attachments can be uploaded as multipart form files, listed as metadata,
and downloaded by attachment ID.

Watchlists are available for compliance and AML-style screening. Admins can
manage account, name, BIC, and keyword entries from the dashboard Configuration
view or through:

```sh
GET /v1/watchlist-entries?include_paused=true&limit=100
POST /v1/watchlist-entries
GET /v1/watchlist-entries/{id}
PATCH /v1/watchlist-entries/{id}
```

Watchlist screening uses exact and normalized matching. Name, account, and BIC
entries match normalized field equality; keyword entries match normalized
remittance text containment. Hits are returned as rule hits and stored in the
transfer decision audit metadata so the reason remains explainable. Fuzzy
matching is not enabled by default.

External sanctions, PEP, adverse-media, or bureau screening can be integrated by
implementing the Go `ComplianceSignalConnector` interface and registering it on
the service. Connector-provided signals flow through the same explainable
watchlist hit and audit path as internal watchlist entries.

Behavior profiles are generated from rolling transfer history and are available
from the Accounts dashboard or:

```sh
GET /v1/accounts/{id}/profile
```

Profiles include usual amount ranges, usual currencies, common creditors,
common debtor/creditor agents, typical active hours, transfer frequency, and
cold-start status. Deviation signals are added as explainable rule hits such as
`behavior-profile-deviation`, and rules can reference fields like
`behavior.amount_deviation_factor`, `behavior.currency_drift`, and
`behavior.profile_cold_start`.

Network analysis is generated from recent transfer relationships and is
available from the Activity dashboard or:

```sh
GET /v1/network/summary
GET /v1/network/summary?account_id=acct-001
```

Network summaries flag fan-in, fan-out, pass-through, circular-flow, rapid
inward-to-outward movement, shared debtor-agent, and shared beneficiary patterns.
Decision-time network hits are returned as `network-mule-pattern`, and rules can
reference fields like `network.score`, `network.rapid_movement`,
`network.fan_in`, and `network.pass_through`.

Security and operations hardening assumes cloud-provided infrastructure for
managed PostgreSQL, managed Redis, TLS/load balancing, WAF/security groups,
centralized logs, metrics, secrets, and backups. Byakugan provides app-level
controls:

- scoped API keys on protected endpoints
- role-aware admin sessions with per-account dashboard module assignments
- request IDs and correlation IDs via `X-Request-ID` and `X-Correlation-ID`
- per-client and admin rate limiting
- structured JSON request logs
- Prometheus-style metrics at `GET /metrics`

Rate limits can be tuned with:

```sh
CLIENT_RATE_LIMIT_PER_MINUTE=120
ADMIN_RATE_LIMIT_PER_MINUTE=60
```

These are fixed-window limits per API token or remote IP. When `REDIS_ADDR` is
set, rate-limit counters are stored in Redis so limits are shared across
application replicas; otherwise Byakugan uses a local in-memory limiter for
dependency-free development. Use lower values for public-facing staging
environments and higher values only after load testing.

Operational guidance is in `docs/operations.md`.
Release acceptance is defined in `docs/release-checklist.md`; compatibility and
performance contracts are in `docs/compatibility.md` and
`docs/performance-budgets.md`.

Investigation and retention endpoints are available for production operations:

```sh
GET /v1/transfer-checks
GET /v1/transfer-checks/export?format=json
GET /v1/transfer-checks/export?format=csv
GET /v1/events
POST /v1/retention/purge
POST /v1/retention/preview
GET /v1/audit-reports/export?category=approvals&format=csv
```

Audit events use complete store-backed pagination with `page` and `page_size`
(maximum 100). Action, entity, entity ID, and date filters are applied before
the matching total is counted and the requested page is read.

Retention previews include dependent rule hits, cases, notes, attachments, and
notification attempts. Case evidence packages are available from
`GET /v1/cases/{id}/evidence?format=zip`. Rule history and rollback use
`GET /v1/rules/{id}/history` and `POST /v1/rules/{id}/rollback`. Rollback always
creates a new immutable version.

Transfer-check and audit-event list endpoints support pagination and filters.
Retention purge supports `dry_run` and is restricted to internal/private clients
with an authenticated admin session.

The dashboard integrates the visual builder directly into the Rules
module at `http://localhost:9096/#rules`. Builder fields, operators, outcomes,
modes, and presets come from `GET /v1/rules/metadata`. Generated definitions can
be edited as JSON, simulated, copied, and sent directly into rule creation
without browser storage.

## Configuration

The dashboard API key input is under the dashboard Configuration menu. Client
API keys can also be created, listed, activated, and deactivated there. An API
key can optionally have an expiry date.

Client API key endpoints:

```sh
GET /v1/api-keys
POST /v1/api-keys
PATCH /v1/api-keys/{id}
```

API key management endpoints are restricted to localhost or private/internal
client addresses and require an admin login session.

Sample create payload:

```json
{
  "name": "Core Banking Client",
  "scopes": ["transfers:write", "transfers:read", "dashboard:read"],
  "expires_at": "2026-12-31T23:59:59Z"
}
```

`POST /v1/api-keys` returns the raw secret only once. Store it immediately and
send it as either `Authorization: Bearer <secret>` or `X-API-Key: <secret>`.
Omit `expires_at` for a non-expiring key.

Runtime settings are managed through DB-driven configuration APIs and can be
cached by TTL:

```sh
GET /v1/configurations
GET /v1/configurations/{key}
PUT /v1/configurations/{key}
POST /v1/configurations/cache/refresh
```

Each configuration entry has a `cache_ttl_seconds` value. Updating an entry
invalidates that entry in cache; the refresh endpoint manually clears cached
configuration values.

Dashboard dropdowns, select choices, filter choices, presets, and defaults must
originate from Byakugan APIs or configuration endpoints rather than hardcoded
HTML or JavaScript lists. Current dashboard-wide choices are DB-driven through
the generic dropdown registry, which the dashboard reads through
`GET /v1/dropdown-items` using `group_id` and `code_id`. Super administrators
maintain codes in the Dropdown Maintenance module. Defaults remain in
`GET /v1/configurations/dashboard.options`; Redis-backed configuration cache can
be used for performance. If a UI workflow needs an option list that the API
does not expose, the API or configuration contract should be extended and
documented instead of duplicating server-owned domain values in the UI.

## Non-Production Simulator

The simulator is a dashboard module and is disabled by default. Enable it only
in development or staging:

```sh
ENABLE_SIMULATOR=true
```

It is available as a runtime-gated module at
`http://localhost:9096/#simulator`. Set both API-side
`ENABLE_SIMULATOR=true` and dashboard-side
`DASHBOARD_SIMULATOR_ENABLED=true`. When either flag is false, normal dashboard
routing cannot open the simulator. Its scenario dropdown and request templates
come from `GET /v1/simulator/metadata`; it uses the current admin session and
never requests, embeds, or stores an API key.

## Offline Licensing

Byakugan requires an Ed25519-signed offline license and counts active API
replicas through PostgreSQL leases. Dashboard, PostgreSQL, and Redis containers
do not consume replica allowances. See [license.md](license.md) for the format,
state model, recovery behavior, and rollout contract, and
[license_generate.md](license_generate.md) for the end-to-end operator workflow.

Generate an offline issuer key outside the product deployment:

```sh
go run ./cmd/byakugan-license keygen \
  --key-id issuer-2027 \
  --private-out issuer.private \
  --public-out issuer-public.json
```

Issue a license from the installation request downloaded through the License
Administration dashboard:

```sh
go run ./cmd/byakugan-license issue \
  --request installation-request.json \
  --customer-id customer_acme_bank_ph \
  --license-id lic_acme_bank_ph_2027 \
  --key-id issuer-2027 \
  --private-key issuer.private \
  --not-before 2027-01-01T00:00:00Z \
  --expires 2028-12-31T23:59:59Z \
  --grace-days 7 \
  --max-api-replicas 4 \
  --features fraud_decisions,dashboard \
  --out byakugan-license.bgl
```

`--customer-id`, `--key-id`, expiry, replica limit, request, private key, and
output are required. The license ID is generated when omitted; activation
defaults to issuance time, grace defaults to seven days, and features default to
`fraud_decisions,dashboard`.

Embed the contents of `issuer-public.json` at image build time through
standard Base64 and pass it as `LICENSE_PUBLIC_KEYS_BASE64` when building the
API image. Start the API, sign the downloaded installation
request with `byakugan-license issue`, then install the `.bgl` file through the
License Administration dashboard. Private issuer keys must never enter product
images or customer runtime environments.

To produce only the customer API executable, without exporting the source,
issuer CLI, Go toolchain, dashboard, or container filesystem, run the dedicated
build script. Create a dedicated artifact-signing key once; do not reuse the
license issuer key:

```sh
openssl genpkey -algorithm ED25519 -out artifact-signing-private.pem
openssl pkey -in artifact-signing-private.pem -pubout -out artifact-signing-public.pem

sh scripts/build-customer-binary.sh \
  issuer-public.json linux/amd64 dist/customer
```

When no artifact key paths are supplied, the script offers to create
`artifact-signing-private.pem` and `artifact-signing-public.pem` in the current
directory. It reuses that pair on later builds and never overwrites a partial
or existing pair. For centrally managed keys, pass the private and public PEM
paths as the fourth and fifth arguments. Production export also refuses an
empty licensing public-key ring, so obfuscation cannot accidentally produce an
image with no trusted license issuer.

If Docker requires root access, run the same command with `sudo`; the script
returns the resulting binary and checksum to the invoking user's ownership:

```sh
sudo sh scripts/build-customer-binary.sh \
  issuer-public.json linux/amd64 dist/customer
```

The customer artifacts are the obfuscated `dist/customer/byakugan-api`, its
`byakugan-api.sha256` checksum, detached `byakugan-api.sig` signature, and
`byakugan-api-signing-public.pem` verification key. The private signing key
never enters Docker or the delivery directory. The build uses pinned Garble
with literal obfuscation, strips local source paths and debugging symbols, and
contains only the trusted public licensing keys. Obfuscation raises reverse-
engineering cost but cannot make executable code impossible to analyze.

Customers verify the signature and checksum with:

```sh
openssl pkeyutl -verify -rawin -pubin \
  -inkey byakugan-api-signing-public.pem \
  -sigfile byakugan-api.sig -in byakugan-api
sha256sum -c byakugan-api.sha256
```

Do not include `issuer.private` or `artifact-signing-private.pem` in `dist` or
any customer delivery package. Pass `linux/arm64` as the second argument for an
ARM64 customer environment, or a third argument to select another output
directory.
If a removed Docker Desktop installation left `credsStore=desktop` configured
without its credential-helper executable, the script uses a temporary anonymous
Docker client configuration for this public-image build. It does not modify
`~/.docker/config.json` or copy registry credentials into the output.

Docker Compose uses the normal stripped API build and does not use Garble or
artifact signing. Obfuscation and detached signing apply only to the dedicated
customer `binary` export target. The private artifact-signing key is never
accepted by Compose or Docker.

## Rules

See [Rule.md](Rule.md) for the rule DSL, default Philippines/PHP rules,
idempotency behavior, scoring, and sample rule commands.

The service does not embed an ML runtime, but external model integrations can
submit versioned `external_scores` containing a normalized score, reason codes,
provenance, and generation time. `advanced_scoring.policy` defaults to shadow
mode; only a fresh configured champion in explicit enforce mode can influence
`external.ml_score` rules. Model names and governance choices are maintained
through the API-owned dropdown registry.

Policy promotion uses versioned `GET /v1/policy-bundles/export` and dry-run-first
`POST /v1/policy-bundles/import`. Behavior profile snapshots and saved simulator
fixtures provide auditable recalibration and regression artifacts.

## Webhooks

Byakugan includes webhook registration APIs for notification targets:

```sh
curl -X POST http://localhost:9095/v1/webhooks \
  -H 'authorization: Bearer dev-secret' \
  -H 'content-type: application/json' \
  -d '{"url":"https://example.test/fraud-events"}'

curl -H 'authorization: Bearer dev-secret' http://localhost:9095/v1/webhooks
```

Current v1 behavior:

- `POST /v1/webhooks` stores an enabled webhook destination.
- `GET /v1/webhooks` lists configured destinations.
- `PATCH /v1/webhooks/{id}` edits a destination URL or enables/disables it.
- Webhook `type` controls routing:
  - `payment_processor` receives ISO/payment release and rejection events.
  - `notification` receives review/block alerts for user intervention.
- Outbound HTTP delivery is active for configured fraud events.
- `GET /v1/notification-attempts` lists recent delivery attempts.
- `POST /v1/notification-attempts/{id}/replay` replays a stored attempt payload.
- Transfer decisions are recorded as `transfer_check.decided` audit events.
- Manual review approvals are recorded as `transfer_check.approved` audit events.
- Confirmed fraud case closures are recorded as `case.resolved_fraud` events so
  an ISO service can reject a held transfer.
- Webhook bodies include the original `transfer_check_request`, the stored
  `transfer_check`, and a canonical JSON `pacs002` status report so downstream
  services do not need to fetch the transfer check again.

- Notification webhooks trigger after a transfer check decision is persisted and
  the decision is included in
  `notifications.policy.value.notify_on_decisions`, which defaults to
  `review` and `block`.
- Payment processor webhooks trigger on manual approval events by default through
  `notifications.policy.value.notify_on_approvals`.
- Payment processor webhooks also receive `case.resolved_fraud` so ISO can
  reject held transfers.
- Store each delivery attempt, response status, and error in
  `notification_attempts`.
- Inject `WEBHOOK_SIGNING_KEY_ID` and `WEBHOOK_SIGNING_SECRET` from the cloud
  secret manager to send `X-Byakugan-Signature` and
  `X-Byakugan-Signature-Key-ID`. Keep the previous key pair configured during
  the receiver rotation window. The legacy configuration secret remains only
  for backward compatibility.

Recommended receiver behavior:

- Use HTTPS URLs reachable from the Byakugan deployment.
- Return a `2xx` response only after the event is accepted.
- Treat events as idempotent by event ID or transfer check ID.
- Do not put secrets in the URL; prefer signature verification or a dedicated
  receiver token.

Example webhook event:

```json
{
  "event": "transfer_check.decided",
  "event_id": "evt-001",
  "transfer_check_id": "chk-001",
  "decision": "review",
  "score": 65,
  "reasons": ["rule: common_php_large_review"],
  "created_at": "2026-07-07T08:30:00Z",
  "transfer_check_request": {
    "message_type": "pacs.008",
    "payload": {
      "message_id": "msg-001",
      "payment_identification": {
        "instruction_id": "inst-001",
        "end_to_end_id": "e2e-001"
      }
    }
  },
  "pacs002": {
    "message_type": "pacs.002",
    "original_message_id": "msg-001",
    "original_message_name_id": "pacs.008",
    "original_instruction_id": "inst-001",
    "original_end_to_end_id": "e2e-001",
    "transaction_status": "PDNG",
    "status_reason": "fraud_review_required",
    "decision": "review",
    "transfer_check_id": "chk-001"
  }
}
```

## Authentication

Set `API_KEYS` to a comma-separated list of bootstrap API keys. If unset and no
DB-managed API keys exist, auth is disabled for local development. Once an active
DB-managed API key exists, requests must use either a bootstrap key or an active
DB-managed key.

The Byakugan admin dashboard does not use client API keys for its own access. It uses the
admin username/password session cookie and can create, list, revoke, and
reactivate client keys.

The standalone dashboard restores identity through `GET /admin/session`, which
returns authentication status, username, role, and assigned modules. Super
administrators assign modules from Configuration → Administrators → Access.
Only assigned modules are added to the sidebar and router, and Go enforces the
same assignment on module API requests. Shared dropdowns and
`dashboard.options` remain readable by every authenticated dashboard user;
their mutation remains restricted. Module changes apply on the user's next
login. Its shared API client clears
expired sessions on `401`, distinguishes permission, timeout, network, and
gateway failures, and handles JSON, multipart uploads, and authenticated file
downloads without storing the session secret in browser storage.

```sh
API_KEYS=dev-secret go run ./cmd/byakugan-api
```

Pass keys with:

```http
Authorization: Bearer dev-secret
```

## Example Transfer Check

```sh
curl -X POST http://localhost:9095/v1/transfer-checks \
  -H 'authorization: Bearer dev-secret' \
  -H 'content-type: application/json' \
  -d @testdata/pacs008-transfer.json
```

## Example ISO 20022 XML Transfer Check

The XML endpoint accepts interoperable ISO 20022 `pacs.008` XML. It searches by
XML local-name, so namespace prefixes such as `ct:` or `p8:` and bank/switch
wrapper names do not need to match a hardcoded layout. It supports direct
`Document/FIToFICstmrCdtTrf` payloads, BancNet profile
`Message/CreditTransfer/FIToFICstmrCdtTrf` envelopes, and other wrappers that
contain `FIToFICstmrCdtTrf`. It maps the first `CdtTrfTxInf` transaction to the
same canonical request used by the JSON endpoint, then runs the normal fraud
decision flow:

```sh
curl -X POST http://localhost:9095/v1/transfer-checks/iso20022 \
  -H 'authorization: Bearer dev-secret' \
  -H 'content-type: application/xml' \
  -d @testdata/pacs008-transfer.xml
```

To test duplicate-payment detection, send the same XML again without
`Idempotency-Key`. The second submission should return `review` with the
`builtin-repeated-inward-transfer` rule hit. If you send the same
`Idempotency-Key`, the second submission is now also flagged for review by
`rule-duplicate-idempotency-key-review`.

Supported XML fields for the first implementation:

- `GrpHdr.MsgId`
- `GrpHdr.CreDtTm`
- `CdtTrfTxInf.PmtId.InstrId`
- `CdtTrfTxInf.PmtId.EndToEndId`
- `CdtTrfTxInf.PmtId.TxId`
- `CdtTrfTxInf.IntrBkSttlmAmt` with `Ccy`, falling back to `InstdAmt`
- `Dbtr.Nm`, `DbtrAcct.Id.IBAN`, `DbtrAcct.Id.Othr.Id`
- `DbtrAgt.FinInstnId.BICFI`, falling back to `InstgAgt.FinInstnId.BICFI`
- `Cdtr.Nm`, `CdtrAcct.Id.IBAN`, `CdtrAcct.Id.Othr.Id`
- `CdtrAgt.FinInstnId.BICFI`, falling back to `InstdAgt.FinInstnId.BICFI`
- `RmtInf.Ustrd`

## ISO 20022 pacs.002 Payment Status Responses

ISO 20022 `pacs.002` responses can be correlated to an existing `pacs.008`
transfer check and stored as operational payment status. The parser supports a
direct ISO `Document`, the supplied BancNet profile envelope, and arbitrary
service wrappers containing `FIToFIPmtStsRpt`:

```sh
curl -X POST http://localhost:9095/v1/payment-statuses/iso20022 \
  -H 'authorization: Bearer dev-secret' \
  -H 'content-type: application/xml' \
  -d @pacs002-response.xml
```

Correlation uses `OrgnlTxId`, then `OrgnlInstrId`, `OrgnlEndToEndId`,
`ClrSysRef`, and finally `OrgnlMsgId`. A new report returns `201`; replaying the
same report is idempotent and returns the stored report with `200`. List reports with
`GET /v1/payment-statuses?transfer_check_id=<check-id>`.

Payment status such as `RJCT`, `ACTC`, or `PDNG` is intentionally separate from
the Byakugan fraud decision. The endpoint detects an XML signature but does not
cryptographically verify XMLDSig; deployments must verify messages at the
trusted ingress boundary before forwarding them to this endpoint. The parser
maps ISO fields into the canonical model but intentionally does not impose a
scheme-specific XSD profile. A payment-service adapter may validate its own XSD
before calling Byakugan.

## View Current Content

With Docker Compose auth enabled, use `dev-secret`:

```sh
curl -H 'authorization: Bearer dev-secret' http://localhost:9095/v1/accounts
curl -H 'authorization: Bearer dev-secret' http://localhost:9095/v1/rules
curl -H 'authorization: Bearer dev-secret' http://localhost:9095/v1/dashboard/decisions
curl -H 'authorization: Bearer dev-secret' http://localhost:9095/v1/dashboard/rule-hits
curl -H 'authorization: Bearer dev-secret' http://localhost:9095/v1/events
```

Default startup rules are installed when the rule store is empty:

- review PHP transfers at or above `50000`
- block PHP transfers at or above `250000`
- review non-PHP transfers because the default deployment profile is Philippines/PHP
- review non-PH debtor or creditor agent BIC patterns
- block high-value non-PHP transfers
- review idempotency keys reused with a different payload
- review duplicate instruction IDs from the same debtor agent
- review repeated inward transfers with the same debtor, creditor, amount, currency, and end-to-end id

Repeated requests with the same `Idempotency-Key` or `idempotency_key` and the
same transfer payload return the original decision. Reusing the same key for a
different payload is treated as a duplicate identity signal and normally returns
`review`. Duplicate `instruction_id` values from the same debtor agent are also
flagged for review.
