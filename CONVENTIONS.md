# Conventions — backoffice-kit

Every module follows these rules. If a rule blocks you, write it in AGENT_LOG.md and ask
in the PR description instead of silently breaking it.

## Stack

| Area            | Choice                                                          |
|-----------------|-----------------------------------------------------------------|
| Runtime         | Node.js 24 LTS, ESM only (`"type": "module"`)                   |
| Language        | TypeScript, `strict: true`, no `any` without a comment why      |
| Package manager | pnpm (version pinned in `packageManager`)                       |
| HTTP            | Express 5 via `@nano-tech-2026-org/core-http` (from M03 on)                |
| Validation      | Zod (env, request bodies, responses where useful)               |
| Database        | PostgreSQL 17 + Drizzle ORM + drizzle-kit migrations            |
| Queue / cache   | Redis 7 + BullMQ                                                |
| Object storage  | S3-compatible (MinIO in dev)                                    |
| Logging         | `@nano-tech-2026-org/core-logger` (pino) — never `console.log` in src      |
| Tests           | Vitest; Supertest for HTTP; Testcontainers for Postgres/Redis   |
| Lint/format     | Biome (`biome check`)                                           |
| Build           | tsup (libs → ESM + d.ts), `tsc --noEmit` for typecheck          |
| Frontend (M12)  | Vue 3 + Quasar + Pinia + TypeScript + Vitest                    |
| Container       | Multi-stage Dockerfile, `node:24-alpine`, non-root user         |

Do not add a dependency that duplicates one of the above. Prefer small, well-maintained
packages; mention every new runtime dependency and why in the PR description.

## Repo layout

```
<module>/
├── src/
│   ├── index.ts          # libs: public API only. services: bootstrap
│   ├── config.ts         # services: Zod env schema via core-config
│   ├── routes/           # services: one file per resource
│   ├── db/               # services with DB: schema.ts, client.ts
│   └── ...
├── migrations/           # drizzle-kit output (services with DB)
├── test/
│   ├── unit/
│   └── integration/      # Testcontainers; may be skipped locally with SKIP_INTEGRATION=1
├── .github/              # from template (CI, release, PR template) — keep
├── .env.example          # every env var, with safe example values
├── Dockerfile            # services only
├── biome.json
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── CHANGELOG.md
└── README.md
```

## package.json

- Name: `@nano-tech-2026-org/<repo-name>`.
- Libraries: `"private": false`, `exports` map, `files: ["dist"]`, `version` starts at `0.1.0`.
- Services/apps: `"private": true` (never published).
- Required scripts (CI calls exactly these):
  `lint` (biome check .), `typecheck` (tsc --noEmit), `test` (vitest run --coverage), `build`.
- Services also: `dev`, `start`, `db:generate`, `db:migrate` (if DB).

## Consuming other modules

- Libraries are consumed from npm at a published version, e.g. `"@nano-tech-2026-org/core-config": "^0.1.0"`.
- If a needed library is merged but **not yet published**, do not work around it with
  file paths or copied code — note it in AGENT_LOG.md and pick another item.
- Services talk to each other over HTTP only, using service-account tokens from auth-service.
  Never connect to another service's database.

## Code rules

- Small pure functions; dependency injection over module-level singletons where testable.
- Errors: throw typed errors (`AppError` from core-http) with a stable `code`; never leak stack traces to clients.
- Config only through the Zod schema; no `process.env` reads outside `config.ts`.
- No secrets in code, tests, fixtures, or logs. Test secrets are generated at runtime.
- SQL only through Drizzle; raw SQL only in migrations or with a comment explaining why.
- Every HTTP endpoint: validation schema, permission check (from M05 on), audit event for
  state-changing actions (from M06 on), test for success + at least one failure case.

## Tests

- Coverage target: ≥ 80% lines for services, ≥ 90% for libraries.
- Unit tests must not need network or Docker.
- Integration tests use Testcontainers; no shared external DB.

## Git & PRs

- Branch: `agent/YYYY-MM-DD-<short-slug>`; maintainer branches: anything else.
- Conventional Commits: `feat:`, `fix:`, `test:`, `docs:`, `refactor:`, `chore:`, `ci:`.
- Several small logical commits per PR, not one giant commit.
- Every commit made by the agent ends with the trailer:
  `Co-authored-by: Hermes Agent <hermes-agent@users.noreply.github.com>`
- PR description: what & why, how to test, checklist from the template, follow-ups.
- Agents never push to `main`, never merge, never force-push to a branch with review comments.

## README per module (required sections)

1. One-line description + badges (CI, npm version for libs)
2. Why / where it fits in backoffice-kit (link to control repo)
3. Install / run (with `.env.example` reference)
4. Usage example (copy-pasteable)
5. API (endpoints table for services, exported functions for libs)
6. Configuration (env table: name, required, default, description)
7. Development (scripts, how to run tests)
8. License

## Versioning & releases

- SemVer. `0.x` while the API is unstable.
- CHANGELOG.md in Keep a Changelog format, updated in every PR under `## [Unreleased]`.
- Releases are created **only by the maintainer** (GitHub Release → CI publishes to npm).

## Ports (local docker-stack)

| Service        | Port |
|----------------|------|
| api-gateway    | 8080 |
| auth-service   | 8081 |
| audit-log      | 8082 |
| notify-service | 8083 |
| file-service   | 8084 |
| report-service | 8085 |
| job-scheduler  | 8086 |
| admin-ui (dev) | 9000 |
