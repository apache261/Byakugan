# Byakugan

**Explainable fraud decisions and AML/CFT monitoring for payment operations.**

Byakugan is a Go-based platform for reviewing bank transfers, investigating
suspicious activity, and giving operations teams a clear view of the evidence
behind each alert. It combines a synchronous fraud-check API with a separate
anti-money-laundering and counter-terrorist-financing (AML/CFT) monitoring
subsystem and a standalone administrative dashboard.

Built around ISO 20022-style payment data, Byakugan is designed for teams that
need configurable controls, traceable outcomes, and practical investigation
workflows.

## How it started

I started Byakugan in 2024 during my free time as a software developer working
with ISO 20022 payments. I wanted to turn the payment data I worked with into
clear, explainable fraud checks and useful tools for the people reviewing them.
The project has since grown to include a separate AML/CFT monitoring and
investigation workflow.

## What Byakugan offers

### Fraud checks at payment time

- Evaluate a transfer synchronously and return an `allow`, `review`, or `block`
  decision with the rule hits that explain it.
- Accept canonical JSON and `pacs.008` XML payment data.
- Configure account controls, thresholds, watchlists, and portable JSON rules.
- Detect patterns involving velocity, new beneficiaries, account behavior,
  unusual routes, and transfer networks.
- Simulate rule changes and inspect decision history before updating policies.
- Manage reviews, fraud cases, notes, attachments, and audit records from the
  dashboard or REST API.

### AML/CFT monitoring and investigations

- Maintain a separate, append-only AML transaction ledger with customer risk
  snapshots and account profiles.
- Run versioned monitoring scenarios that produce explainable alerts and
  evidence without changing payment authorization decisions.
- Investigate alerts in confidential AML cases, attach evidence, and prepare
  regulator-neutral suspicious-activity report packages with independent
  approval.
- Import historical data through mapped CSV, JSON, NDJSON, or `pacs.008`
  sources, then run separately authorized monitoring backfills.
- Apply legal holds and retention controls to protected AML records.

### An operations-focused platform

- Use a standalone, role-restricted dashboard for fraud and AML workflows.
- Integrate through documented REST APIs, OpenAPI, and configurable webhooks.
- Run locally with an in-memory store, or use PostgreSQL persistence and
  optional Redis caching for a deployed environment.
- Access request tracing, health checks, metrics, and operational runbooks.

## Architecture at a glance

```text
Payment integration ──> Fraud-check API ──> allow / review / block
                             │
                             └──> fraud evidence and case workflows

Transaction sources ──> AML ledger ──> monitoring ──> alerts and investigations

                     Standalone admin dashboard
```

Fraud checks and AML monitoring have separate records, permissions, and
workflows. AML alerts inform investigations; they do not authorize, reject, or
hold payments. The core is jurisdiction-neutral, so institution-specific lists,
reporting formats, thresholds, and submission connectors are configured or
integrated for the target environment.

## Project access

Further development of Byakugan has moved to a private repository. For project
information or access inquiries, contact
[lynolibarra@gmail.com](mailto:lynolibarra@gmail.com).
