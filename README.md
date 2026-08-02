# Observability Playbook

[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE) [![Web Reactions](https://api.webreactions.app/badge/github/khasky/observability-playbook.svg)](https://webreactions.app/?utm_source=github&utm_channel=repository&utm_medium=observability-playbook)

Practical guide to production visibility: structured logs, distributed tracing, metrics, SLOs, and alerting that respects humans.

> *If I were instrumenting a production system today, what would I standardize before the first serious incident forced me to?*

This playbook is about:
- structured logging that survives an incident;
- distributed tracing across services and queues;
- metrics that describe users, not just machines;
- SLOs and error budgets as working agreements;
- alerting that respects the humans on call;
- dashboards that answer questions instead of decorating walls;
- correlation from the edge to the last background job.

---

## Table of Contents

- [Observability Playbook](#observability-playbook)
  - [Table of Contents](#table-of-contents)
  - [Companion playbooks](#companion-playbooks)
  - [Philosophy](#philosophy)
  - [The defaults I'd reach for first](#the-defaults-id-reach-for-first)
  - [Structured logging rules](#structured-logging-rules)
    - [Levels with meaning](#levels-with-meaning)
    - [The canonical log line](#the-canonical-log-line)
    - [What never goes in a log](#what-never-goes-in-a-log)
    - [Sampling](#sampling)
  - [Tracing](#tracing)
    - [Span discipline](#span-discipline)
    - [What usually goes wrong](#what-usually-goes-wrong)
  - [Metrics](#metrics)
    - [My default view](#my-default-view)
    - [Cardinality discipline](#cardinality-discipline)
  - [SLOs and error budgets](#slos-and-error-budgets)
  - [Alerting](#alerting)
    - [Page or ticket](#page-or-ticket)
  - [Dashboards](#dashboards)
  - [Correlation end to end](#correlation-end-to-end)
  - [Things to avoid](#things-to-avoid)
  - [Worth reading](#worth-reading)
  - [License](#license)

---

## Companion playbooks

These repositories form one playbook suite:

- [AI-Assisted Engineering Playbook](https://github.com/khasky/ai-assisted-engineering-playbook) — agent workflows, guardrails, and quality control for AI-heavy teams
- [API Design Playbook](https://github.com/khasky/api-design-playbook) — versioning, pagination, idempotency, error contracts, and webhooks
- [Auth & Identity Playbook](https://github.com/khasky/auth-identity-playbook) — sessions, tokens, OAuth, and identity boundaries across the stack
- [Backend Architecture Playbook](https://github.com/khasky/backend-architecture-playbook) — APIs, boundaries, OpenAPI, persistence, and errors
- [Best of JavaScript](https://github.com/khasky/best-of-javascript) — curated JS/TS tooling and stack defaults
- [Caching Playbook](https://github.com/khasky/caching-playbook) — HTTP, CDN, and application caches; consistency and invalidation
- [Code Review Playbook](https://github.com/khasky/code-review-playbook) — PR quality, ownership, and review culture
- [CTO Playbook](https://github.com/khasky/cto-playbook) — org design, hiring, technical strategy, budgets, and due diligence
- [DevOps Delivery Playbook](https://github.com/khasky/devops-delivery-playbook) — CI/CD, environments, rollout safety, and observability
- [Engineering Lead Playbook](https://github.com/khasky/engineering-lead-playbook) — standards, RFCs, and technical leadership habits
- [Frontend Architecture Playbook](https://github.com/khasky/frontend-architecture-playbook) — React structure, performance, and consuming API contracts
- [Git Collaboration Playbook](https://github.com/khasky/git-collaboration-playbook) — branching, stacked PRs, merge queues, and CI collaboration at scale
- [Marketing and SEO Playbook](https://github.com/khasky/marketing-and-seo-playbook) — growth, SEO, experimentation, and marketing surfaces
- [Messaging & Async Playbook](https://github.com/khasky/messaging-and-async-playbook) — queues, events, outbox, idempotent consumers, and retries
- [Monorepo Architecture Playbook](https://github.com/khasky/monorepo-architecture-playbook) — workspaces, package boundaries, and shared code at scale
- [Node.js Runtime & Performance Playbook](https://github.com/khasky/nodejs-runtime-performance-playbook) — event loop, streams, memory, and production Node performance
- **Observability Playbook** — logs, traces, metrics, SLOs, and production visibility
- [React Cross-Platform Playbook](https://github.com/khasky/react-cross-platform-playbook) — shared React UI and logic across web and native with TypeScript
- [Software Design Playbook](https://github.com/khasky/software-design-playbook) — separation of concerns, composition, and module boundaries
- [State Management Playbook](https://github.com/khasky/state-management-playbook) — client state architecture, MobX, and choosing a state layer
- [Styling Architecture Playbook](https://github.com/khasky/styling-architecture-playbook) — type-safe styling, design tokens, and theming at scale
- [Sysadmin Operations Playbook](https://github.com/khasky/sysadmin-operations-playbook) — servers, backups, DNS, TLS, identity, and the self-hosted ops stack
- [Testing Strategy Playbook](https://github.com/khasky/testing-strategy-playbook) — unit, integration, contract, E2E, and CI-friendly test layers

---

## Philosophy

Observability is not a wall of dashboards.
It is being able to ask new questions of a running system without shipping new code.

Most observability work is not about collecting more data. It is about:
- instrumenting for the incident you have not had yet;
- one consistent way to correlate a request across everything it touched;
- separating "a human should look at this" from background noise;
- measuring what users feel, not just what servers do;
- paying only for signals someone actually queries.

Dashboards answer yesterday's questions.
Instrumentation lets you ask tomorrow's.

---

## The defaults I'd reach for first

If I were standing up observability for a product team today, I would start with:

- **Logs:** structured JSON from day one, one shared schema across services
- **Traces:** OpenTelemetry SDK with W3C trace context propagation
- **Correlation:** `request_id` and trace id on every log line and every error response — the same convention that already runs through the rest of this suite
- **Metrics:** RED per service (rate, errors, duration), histograms for latency
- **Alerts:** symptom-based only, driven by SLO burn rather than static thresholds
- **Dashboards:** one per service, with a drill-down path to traces

None of these require exotic tooling.
All of them require deciding before the incident, not during it.

---

## Structured logging rules

### Levels with meaning

Levels are a contract with the reader, not decoration:

- `error` — a human should look at this, ideally soon
- `warn` — degraded but coping; worth a trend line, not a page
- `info` — state changes you would want when reconstructing an incident
- `debug` — development only; off in production

If `error` fires a hundred times an hour and nobody looks, the level is lying.

### The canonical log line

One wide, structured event per request beats twenty scattered lines.

Emit a single canonical log line when the request completes: route, status, duration, `request_id`, trace id, user tier, feature flags, downstream call counts. Everything you would want in one row when the pager goes off.

```json
{"level":"info","msg":"request completed","request_id":"req_8fZk21","trace_id":"4bf92f35","route":"POST /orders","status":201,"duration_ms":142}
```

Scattered incremental logging is how you end up grepping five lines to reconstruct one request.

### What never goes in a log

- secrets, tokens, passwords — absolutely and without exception
- PII — allowlist fields at the logging edge; a denylist always misses one
- full request or response bodies "just in case"

The allowlist lives in one place, at the boundary where logs are emitted.
Every service inventing its own redaction is how leaks happen.

### Sampling

High-volume paths do not need every success recorded:

- keep every error and every slow request;
- sample the healthy majority at a rate you can afford;
- make the sample rate a config value, not a code change.

---

## Tracing

OpenTelemetry SDK plus auto-instrumentation is the floor, not the finish line. Auto-instrumentation gets you HTTP, database, and client spans for free; manual spans mark the operations your business actually cares about.

### Span discipline

- span name = the operation: `GET /orders/:id`, `orders.charge`
- attributes = the dimensions: tenant, region, result, retry count
- never encode values into span names — that is what attributes are for, and dynamic names destroy grouping

### What usually goes wrong

- context dies at the queue boundary — propagate trace context through message headers, not just HTTP; the mechanics are covered in the [Messaging & Async Playbook](https://github.com/khasky/messaging-and-async-playbook)
- traces exist but nobody queries them, so nobody notices they are broken
- 100% sampling on day one, then a bill, then 0% sampling

My preference on sampling: head-based to start — cheap and predictable.
Tail-based when money allows — it keeps the slow and failed traces that head-based throws away.

---

## Metrics

### My default view

- **RED for services:** rate, errors, duration — per service, per endpoint
- **USE for resources:** utilization, saturation, errors — for hosts, pools, queues; deeper runtime diagnostics live in the [Node.js Runtime & Performance Playbook](https://github.com/khasky/nodejs-runtime-performance-playbook)
- **Histograms for latency:** percentiles, not averages — p50 tells you the typical case, p99 tells you who is suffering, the average tells you neither
- **Business metrics next to system metrics:** signups, orders, and payment failures on the same dashboards as 5xx rates — a healthy service with zero orders is an incident

The same RED metrics gate canary rollouts; that loop is covered in the [DevOps Delivery Playbook](https://github.com/khasky/devops-delivery-playbook).

### Cardinality discipline

Labels are for bounded dimensions: status class, region, endpoint template.

- no user ids in labels
- no request ids, emails, or raw URLs in labels
- every unbounded label is a time-series explosion arriving on a delay

If you need per-user answers, that is a trace or a log query, not a metric.

---

## SLOs and error budgets

Pick SLIs users can feel:

- availability — the fraction of requests that succeed;
- latency at a percentile — "p99 under 500 ms", not "average under 100 ms".

Then:

1. **An SLO is an agreement, not a decoration.** Product and engineering both sign it; it prices reliability against feature speed.
2. **The error budget is spending money.** Budget left — ship faster, take risks. Budget gone — reliability work wins the sprint.
3. **Alert on budget burn rate, not raw thresholds.** A fast burn pages; a slow burn opens a ticket. Static thresholds page at 3am for problems that could wait until Tuesday.
4. **SLOs end the "100% uptime" fantasy.** 100% is not a target; it is a refusal to make tradeoffs. The last nine costs more than the product earns.

---

## Alerting

1. **Page on symptoms, ticket on causes.** Users seeing errors is a page. A disk filling over three weeks is a ticket.
2. **Every page is actionable and owned.** If the response to a page is "acknowledge and go back to sleep", delete the alert.
3. **Alert fatigue is an outage generator.** Every ignored page trains the on-call to ignore the one that matters.
4. **Runbooks linked from the alert itself.** The person paged at 3am should be one click from "what this means and what to do", not searching a wiki.

### Page or ticket

- page: SLO burn rate, user-facing error spikes, hard dependency down
- ticket: certificate expiring in three weeks, single host degraded behind a healthy pool, slow disk growth
- neither: a metric wiggled

---

## Dashboards

One dashboard per service, answering two questions: is it healthy, and who is it hurting.

- top row: the SLIs — request rate, error rate, latency percentiles
- below: the usual suspects — saturation, dependencies, queue depth
- every panel drills down: symptom to dashboard, dashboard to trace, trace to logs — all joined by the trace id

A wall of forty graphs is a museum, not a tool.
If a panel has never changed a decision, remove it.

---

## Correlation end to end

The single highest-leverage convention in this suite: one `request_id`, born at the edge, carried everywhere.

- generated (or accepted) at the edge, attached to the trace context;
- stamped on every log line the request produces, in every service;
- echoed in every error envelope, so a user's screenshot leads straight to the trace — the envelope shape is defined in the [API Design Playbook](https://github.com/khasky/api-design-playbook) and wired through services in the [Backend Architecture Playbook](https://github.com/khasky/backend-architecture-playbook);
- carried through queues, jobs, and events, so background work stays attributable — see the [Messaging & Async Playbook](https://github.com/khasky/messaging-and-async-playbook).

A request that becomes three events and a retry is still one story.
Correlation is what lets you read it.

---

## Things to avoid

- unstructured printf logging that only a human with grep can read;
- `error`-level noise nobody reads, burying the one line that mattered;
- averages hiding tail pain while p99 users quietly churn;
- an alert on every metric "to be safe";
- per-request `debug` logging left on in production;
- metrics with unbounded labels melting the metrics backend;
- tracing you pay for but never query.

---

## Worth reading

- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [OpenTelemetry: context propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Google SRE book: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Google SRE book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Google SRE workbook: Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Prometheus: histograms and summaries](https://prometheus.io/docs/practices/histograms/)

---

## License

MIT is a sensible default for a repository like this, but choose the license that fits how you want others to reuse the material.
