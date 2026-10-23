# Roadmap — backoffice-kit

Building blocks for enterprise back-office applications: auth, RBAC, audit, notifications,
files, reports, scheduling, gateway, and an admin dashboard — composable, tested, one-command deploy.

## Status values

`todo` → `in-progress` → `review` (PR open) → `done` (merged to `main`)

A dependency is satisfied **only** when its status is `done`.
Large modules may be delivered in several PRs ("part 1/3", …); the module stays `in-progress`
between parts and becomes `done` only after the last part is merged.

## Modules

| ID  | Module               | Repo           | Type    | Depends on                    | Status | PR |
|-----|----------------------|----------------|---------|-------------------------------|--------|----|
| M01 | Config loader        | core-config    | lib     | –                             | todo   | –  |
| M02 | Structured logger    | core-logger    | lib     | M01                           | todo   | –  |
| M03 | HTTP app foundation  | core-http      | lib     | M01, M02                      | todo   | –  |
| M04 | Auth service         | auth-service   | service | M03                           | todo   | –  |
| M05 | RBAC middleware      | rbac           | lib     | M04                           | todo   | –  |
| M06 | Audit log            | audit-log      | service | M05                           | todo   | –  |
| M07 | Notifications        | notify-service | service | M05                           | todo   | –  |
| M08 | File storage         | file-service   | service | M05                           | todo   | –  |
| M09 | Reports (PDF/Excel)  | report-service | service | M08                           | todo   | –  |
| M10 | Job scheduler        | job-scheduler  | service | M07, M09                      | todo   | –  |
| M11 | API gateway          | api-gateway    | service | M04, M06, M07, M08, M09, M10  | todo   | –  |
| M12 | Admin dashboard      | admin-ui       | app     | M11                           | todo   | –  |
| M13 | One-command stack    | docker-stack   | infra   | M11, M12                      | todo   | –  |
| M14 | Observability        | observability  | infra   | M13                           | todo   | –  |
| M15 | End-to-end tests     | e2e-tests      | test    | M13                           | todo   | –  |
| M16 | Documentation site   | docs-site      | docs    | M13                           | todo   | –  |

## Acceptance criteria

### M01 core-config
- `loadConfig(schema)` built on Zod; reads `process.env` and optional `.env`.
- Fully typed result; fails fast with one readable error listing **all** invalid/missing vars.
- Defaults, coercion (number/boolean/url), and `maskSecrets()` for safe logging.
- Coverage ≥ 90%.

### M02 core-logger
- pino-based; JSON in production, pretty output in development.
- Child loggers, log levels from config, redaction of `password`, `token`, `authorization`, `cookie`.
- `requestId` context via AsyncLocalStorage, automatically attached to every log line.

### M03 core-http
- `createApp({ name, version, routes, readinessChecks })` on Express 5.
- Middleware: request-id (propagates `x-request-id`), access log, Zod `validate({ body, query, params })`.
- Standard error format: `{ "error": { "code", "message", "details", "requestId" } }`.
- `/health` (liveness) and `/ready` (pluggable checks, e.g. DB/Redis).
- Graceful shutdown; OpenAPI generation from Zod schemas, served at `/docs`.

### M04 auth-service (expected 2–3 parts)
- PostgreSQL + Drizzle ORM with migrations: users, roles, permissions, role_permissions, user_roles, refresh_tokens, service_accounts.
- Login, refresh-token rotation with reuse detection, logout, `GET /me`, password change.
- Password hashing with argon2; login rate limiting.
- RS256 access tokens with permissions in claims; JWKS at `/.well-known/jwks.json`.
- Client-credentials grant for service accounts (service-to-service calls).
- Admin endpoints for users/roles/permissions; seed script creates the first admin.
- Integration tests against real Postgres.

### M05 rbac
- `authenticate({ jwksUrl })` middleware using `jose` with JWKS caching.
- `requirePermission('report:create')`, `requireAnyPermission([...])`, wildcard support (`report:*`).
- Typed `req.user`; tests using a locally generated JWKS.

### M06 audit-log
- Append-only events: actor, action, resource type/id, before/after diff, ip, requestId, timestamp.
- `POST /events` (service accounts only), `GET /events` with filters + cursor pagination, CSV export.
- Small client exported from the package (`audit-log/client`) for other services.

### M07 notify-service
- Channels: Telegram Bot API, email (SMTP via nodemailer; Mailpit in dev), webhook (HMAC-signed).
- Handlebars templates, BullMQ queue with retries/backoff, idempotency key, delivery-status API.

### M08 file-service
- S3-compatible storage (MinIO in dev) via `@aws-sdk/client-s3`.
- Presigned upload/download URLs; metadata in Postgres (owner, size, mime, sha256).
- Size limit and MIME allowlist from config; soft delete.

### M09 report-service
- Async report jobs: PDF (pdfmake) and Excel (exceljs) from templates.
- Output stored through file-service; job status endpoint; sample "user list" report.

### M10 job-scheduler
- BullMQ repeatable jobs; CRUD for schedules with validated cron expressions.
- Job types: `send-notification`, `generate-report`, `http-call`; run history; manual trigger.

### M11 api-gateway
- Single entry point routing `/api/<service>/*` to services.
- JWT verification at the edge, Redis-backed rate limiting per user/IP, CORS, request-id propagation.
- Aggregated `/health` and combined OpenAPI docs.

### M12 admin-ui (expected 3–4 parts)
- Vue 3 + Quasar + Pinia + TypeScript, talks only to api-gateway.
- Login with silent refresh; users & roles management; audit log viewer; file manager;
  report generator; schedules; notification test page.
- Permission-aware menus and routes; i18n (Indonesian/English); Vitest component tests.

### M13 docker-stack
- docker-compose running every service plus Postgres, Redis, MinIO, Mailpit.
- `make up`, `make seed`, `make down`; `.env.example`; README quickstart that works in under 5 minutes.

### M14 observability
- Add `/metrics` (prom-client) to core-http as a minor release and adopt it in services.
- Prometheus, Grafana (provisioned dashboards), Loki + Promtail; example alert rules.

### M15 e2e-tests
- Playwright flow: login → create user → assign role → upload file → generate report → see audit entry.
- Runs in GitHub Actions against docker-stack.

### M16 docs-site
- VitePress: architecture overview (with diagrams), per-service pages, ADRs, getting started.
- Deployed to GitHub Pages.

## Phase 2 backlog (not started until moved into the table above by the maintainer)
- 2FA (TOTP) and OAuth login (Google)
- Multi-tenancy
- Feature-flag service
- Typed SDK generated from OpenAPI
- k6 load tests
- Helm chart / Kubernetes manifests
