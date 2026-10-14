# Freight Exchange with Autonomous Quoting

A freight exchange for road transport. Shippers publish loads, and carriers find them and quote, by hand or automatically through pricing rules. It is a learning project: a small-scale version of a real carrier platform. It covers a modern TypeScript stack end to end: an event-driven backend, GraphQL, OAuth 2.0 and OpenID Connect, Kubernetes, and a secure development lifecycle.

> **Status:** in progress, built phase by phase. The full specification, including the architecture, the data model, every technical decision (30 ADRs) and the phase plan, is in [`docs/design.md`](docs/design.md).

---

## Table of contents

- [What it does](#what-it-does)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Key design decisions](#key-design-decisions)
- [Identity and security](#identity-and-security)
- [Secure development lifecycle (SSDLC)](#secure-development-lifecycle-ssdlc)
- [Repository layout](#repository-layout)
- [Getting started](#getting-started)
- [Scripts](#scripts)
- [Testing](#testing)
- [Kubernetes (kind)](#kubernetes-kind)
- [Roadmap](#roadmap)
- [Out of scope](#out-of-scope)

---

## What it does

**For shippers**
- Publish loads with origin, destination, pickup window, weight, vehicle type and a reference price.
- See every quote received for their loads, and award the load to one of them. Awarding a load automatically rejects the other quotes.
- Withdraw a load or let it expire automatically when its pickup window closes.

**For carriers**
- Browse the **load board**, a fast, filterable, cursor-paginated view of open loads. It never shows competitors' prices.
- Submit or withdraw quotes by hand, with a **suggested price** calculated from their own rules.
- Define **quotation rules** (lanes, vehicle types, price per km, minimums and margins). The quotation engine applies them **automatically** when a new load that matches them is published.

**For everyone**
- **Real-time updates.** New loads, quotes and awards appear without reloading, through GraphQL subscriptions.
- Notifications with an unread counter.
- **Multi-tenant B2B.** Each company only ever sees its own data, and this is enforced down to the database.
- Sign-in through the project's own OpenID Connect provider. The browser never sees a token.
- Accessible, keyboard-navigable UI.

## Architecture

```
 Browser (React SPA)
        │  HTTPS, HttpOnly session cookie only
        ▼
 ┌─────────────┐   OIDC (code + PKCE,     ┌──────────────────────┐
 │     BFF     │◄─ private_key_jwt) ─────►│ OpenID provider      │
 │  (Koa)      │                          │ (Koa + node:crypto)  │
 └──────┬──────┘                          └──────────────────────┘
        │  access token (aud = API)
        ▼
 ┌─────────────┐  write model + outbox   ┌──────────────┐
 │  GraphQL    │────────────────────────►│  PostgreSQL  │
 │  API (Koa + │◄──── read model ────────│  (RLS)       │
 │  Apollo)    │                         └──────┬───────┘
 └──────┬──────┘                                │ outbox relay
        │ subscriptions                         ▼
        │                               ┌──────────────┐
        └──────────────────────────────►│    Kafka     │
                                        │  (Redpanda)  │
                                        └──────┬───────┘
              ┌────────────────────────────────┼──────────────────────┐
              ▼                                ▼                      ▼
     quotation-worker                  board-projector       notification-worker
     (automatic quotes)                (CQRS read model)     (notifications)
```

| Component | Role |
|---|---|
| `web` | React SPA. It only ever talks to the BFF. |
| `bff` | Backend for Frontend. It is the confidential OAuth client, holds the tokens in server-side sessions, and proxies GraphQL and WebSockets to the API. |
| `auth` | OpenID Connect provider written from scratch, with discovery, JWKS, authorization, login, token rotation, revocation and logout. |
| `api` | GraphQL API (queries, mutations and subscriptions), REST health and metrics endpoints, outbox relay and scheduled jobs. |
| `quotation-worker` | Consumes load events and produces automatic quotes from the carriers' rules. |
| `board-projector` | Maintains the denormalized load board read model (light CQRS). It can be rebuilt from scratch. |
| `notification-worker` | Turns domain events into user notifications. |

## Tech stack

| Area | Technologies |
|---|---|
| Language and runtime | TypeScript 6 (strict, ESM), Node.js 24 LTS, pnpm 12 workspaces (monorepo) |
| HTTP and API | Koa 3, Apollo Server 5, GraphQL (schema first, SDL), graphql-ws, GraphQL Code Generator, DataLoader |
| Data | PostgreSQL 18, TypeORM 1.x, migrations, Row-Level Security, UUIDv7 |
| Events | Kafka protocol on Redpanda (with an Apache Kafka profile), `@confluentinc/kafka-javascript`, Zod 4 event contracts |
| Frontend | React 19, Vite 8, React Router, Apollo Client 4, vanilla-extract |
| Identity | OAuth 2.0, OpenID Connect, PKCE, `private_key_jwt`, JWT/JWK on `node:crypto` only, Argon2id |
| Testing | Vitest, fast-check (property-based tests), Testcontainers, Playwright with axe-core, k6 |
| Observability | OpenTelemetry, pino, Prometheus metrics, Jaeger v2 |
| Build and deploy | tsdown, Docker (multi-stage), Docker Compose, Kubernetes on kind, Gateway API with NGINX Gateway Fabric, HPA |
| CI and security | GitHub Actions, gitleaks, Semgrep CE, `pnpm audit`, zizmor, actionlint, ZAP, Trivy, Renovate |

## Key design decisions

- **Transactional outbox.** Every state change writes its event in the same transaction, and a relay publishes it to Kafka. No event is lost or published without its change.
- **Idempotent, order-tolerant consumers.** Events carry the full aggregate state and a version, so duplicates and late, out-of-order events are harmless.
- **Failure classification.** Transient errors are retried with backoff and jitter. Errors that can resume later, such as the database being down, pause the consumer. Terminal errors go to a dead letter queue (DLQ) and can be replayed.
- **Light CQRS for the load board only.** The busiest query reads a denormalized model kept up to date by a projector. Everything else reads the write model. There is no event sourcing.
- **Concurrency through conditional updates.** Races such as accept against withdraw, award against expire, and quote while awarding are resolved in the database: exactly one operation wins.
- **Read-your-writes.** Mutations return the entity, the UI is optimistic, and subscriptions keep the eventually consistent board in sync.
- **Zero-downtime schema changes** with the expand/contract pattern.
- **Business errors as values** (GraphQL result unions). Resources of another company respond as non-existent, which prevents IDOR (insecure direct object reference) attacks.

Each decision is recorded as an ADR (architecture decision record) in the design document, with its context, alternatives and trade-offs.

## Identity and security

- **BFF pattern**, as recommended for browser apps. Tokens stay on the server. The browser only has a `__Host-Http-` cookie with `HttpOnly`, `Secure` and `SameSite=Strict`.
- **Authorization code flow with PKCE** and checks on `state`, `nonce` and `iss`. The BFF authenticates to the provider with `private_key_jwt`, using single-use assertions.
- **Short-lived access tokens** (5 minutes), scoped to the API audience. **Rotating refresh tokens** with reuse detection, which revokes the whole token family. Refreshes are single-flight, also across BFF replicas.
- **Signing-key rotation** with `kid` and JWKS, without invalidating live tokens.
- **Layered authorization:** scope, role, company type and tenant. PostgreSQL Row-Level Security fails closed.
- **CSRF protection** with `SameSite`, an anti-CSRF header and `Origin` checks, including on WebSockets.
- **No auth libraries.** JWT, JWK, PKCE, encryption and password hashing are built on `node:crypto` primitives and tested against RFC test vectors and known attacks. This is for learning; production would use a managed identity provider.

## Secure development lifecycle (SSDLC)

Security is integrated into every phase rather than audited at the end. The design follows the NIST SSDF (Secure Software Development Framework):

- **Requirements:** OWASP ASVS 5.0 (Application Security Verification Standard) at level 2, data classification and abuse cases.
- **Design:** a STRIDE threat model with trust boundaries, versioned in `docs/security/`.
- **Implementation:**
  - Custom Semgrep rules for SQL interpolation, insecure cookies and secrets in logs.
  - ESLint security rules.
  - A gitleaks pre-commit hook.
  - A pull request (PR) security checklist.
- **Supply chain:**
  - pnpm with a minimum release age and approved install scripts only.
  - The `no-downgrade` trust policy.
  - Registry signature checks.
  - Renovate for updates.
  - GitHub Actions pinned by commit SHA.
- **Verification in CI:**
  - SAST (static analysis) with Semgrep, and SCA (dependency analysis).
  - Secret scanning and workflow linting.
  - Property-based tests.
  - DAST (testing the running app) with ZAP.
  - Image and manifest scanning with Trivy, plus an SBOM (software bill of materials).
- **Deployment:**
  - Pod Security Standards.
  - Default-deny NetworkPolicies.
  - Non-root containers with read-only file systems.
  - Strict security headers (CSP, HSTS).
- **Operations:** vulnerability SLAs (deadlines by severity) using CVSS, EPSS and the CISA KEV catalogue, incident runbooks and drills.

Report vulnerabilities as described in [`SECURITY.md`](SECURITY.md).

## Repository layout

```
apps/
  api/                  GraphQL API, outbox relay, scheduled jobs
  auth/                 OpenID Connect provider
  bff/                  Backend for Frontend
  quotation-worker/     automatic quotes
  board-projector/      load board read model
  notification-worker/  notifications
  web/                  React SPA
packages/
  domain/               business rules, state machines, quotation engine
  db/                   DataSource, entities, migrations, seeds, withTenant
  events/               Zod event contracts and topic names
  kafka/                producer and consumer (retries, pause, DLQ)
  oidc/                 JWT, JWK, PKCE, encryption and passwords on node:crypto
  graphql-schema/       shared SDL
  observability/        logger, traces, metrics
  config/               validated environment variables
tools/dev-token/        development access token CLI
infra/                  Docker Compose, Postgres roles, kind and k8s manifests, k6
e2e/                    Playwright tests
docs/                   design document, phase notes, security artifacts
```

## Getting started

**Prerequisites:** Node.js 24, pnpm 12, Docker, [mkcert](https://github.com/FiloSottile/mkcert) and [gitleaks](https://github.com/gitleaks/gitleaks). Install mkcert and gitleaks once per machine.

```bash
mkcert -install          # local certificate authority for HTTPS
pnpm install
pnpm certs               # certificates for localhost
pnpm infra:up            # PostgreSQL and Redpanda
pnpm db:migrate
pnpm auth:keys           # provider signing keys and BFF client key
pnpm kafka:topics
pnpm db:seed             # demo companies, users and loads
pnpm dev
```

Then open **https://localhost:5173** and sign in with one of the seeded users.

HTTPS is used locally too, because `__Host-` cookies require it and because development should behave like production.

| Service | Port |
|---|---|
| web (Vite, HTTPS) | 5173 |
| auth (OpenID provider, HTTPS) | 4001 |
| bff | 4002 |
| api (internal only) | 4000 |
| workers (health and metrics) | 4101–4103 |
| PostgreSQL | 5432 |
| Redpanda (Kafka) / Console | 19092 / 8080 |
| Jaeger | 16686 |

## Scripts

| Script | What it does |
|---|---|
| `pnpm dev` | Runs every app with hot reload |
| `pnpm infra:up` / `infra:down` | Starts or stops Docker Compose |
| `pnpm db:migrate` / `db:seed` / `db:seed:bulk` | Runs migrations / loads demo data / loads bulk data |
| `pnpm auth:keys` / `auth:rotate-key` / `auth:token` | Creates keys / rotates the signing key / gets a development access token |
| `pnpm kafka:topics` / `kafka:dlq-replay <topic>` | Creates topics / replays a DLQ |
| `pnpm codegen` | Runs GraphQL Code Generator |
| `pnpm lint` / `typecheck` | Runs ESLint / `tsc --noEmit` |
| `pnpm test` / `test:int` / `e2e` | Runs unit / integration / end-to-end tests |
| `pnpm security` | Runs gitleaks, Semgrep and `pnpm audit` locally |
| `pnpm build` | Builds every app |

## Testing

| Level | Tooling | Covers |
|---|---|---|
| Unit | Vitest | Domain rules, state transitions, quotation engine, RFC vectors for the OIDC code |
| Properties | fast-check | Input parsers (JWT, cursors, cookies, hashes) only ever fail with their controlled error |
| Contract | Vitest + Zod | Every event matches its topic schema |
| Integration | Vitest + Testcontainers | Migrations, RLS, outbox relay, consumers (duplicates, disorder, DLQ, pause), projector rebuild, identity flows |
| API | Apollo `executeOperation` + HTTP | Resolvers, authorization matrix, IDOR, races |
| Frontend | Testing Library + `MockedProvider` | Components and hooks |
| E2E | Playwright + axe-core | Full flows, including sign-in and accessibility |
| Load | k6 | Load board and publishing under sustained traffic |

## Kubernetes (kind)

Everything runs inside a local kind cluster:

- PostgreSQL and Redpanda as StatefulSets, plus Jobs for migrations and topic creation.
- Probes and a HorizontalPodAutoscaler for the quotation worker.
- Browser access through **Gateway API** (NGINX Gateway Fabric) on local `*.localhost` HTTPS hostnames for the web app and the identity provider.
- Hardening: Pod Security Standards, default-deny NetworkPolicies, no service account tokens, and non-root, read-only containers.

## Roadmap

| Phase | Content |
|---|---|
| 0 | Monorepo, CI, Compose, security baseline |
| 1 | Domain and data (TypeORM, migrations, RLS) |
| 1b | Identity: OpenID provider, BFF, local HTTPS |
| 2 | GraphQL API: authorization, pagination, DataLoaders, races, outbox |
| 3 | Events: relay, workers, scheduled jobs, DLQ |
| 3b | CQRS read model for the load board |
| 4 | React frontend with Apollo Client |
| 5 | Real time: subscriptions through the BFF |
| 6 | Observability, performance, load tests, E2E, DAST |
| 7 | Docker images and Kubernetes on kind |
| 8 | Optional extras: Elasticsearch, delayed retry topics, Grafana, passkeys |

## Out of scope

The project leaves out the following:

- Event sourcing and a schema registry.
- Real email or push delivery.
- User registration.
- Payments and multiple currencies.
- Cloud deployment.

It focuses on the core flows and on doing them properly.
