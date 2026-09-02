<!--
Sync Impact Report
===================
Version change: 1.1.0 → 1.2.0
Rationale: MINOR bump — materially expanded guidance within an existing
principle (I. API Contract & RESTful Design): added a mandatory timestamp
format standard for all timestamp input/output, plus a cross-reference from
Principle IV so structured log timestamps follow the same standard. No
principle was removed or redefined in a backward-incompatible way.

Modified principles:
  - I. API Contract & RESTful Design — added required timestamp format
    (`YYYY-MM-DDTHH:mm:ss±H:mm`, e.g. `2026-09-02T14:30:00+7:00`) for all
    timestamp values in API requests and responses.
  - IV. Explicit Error Handling & Observability — added a cross-reference
    requiring structured log timestamps to use the same format defined in
    Principle I.
Added sections: none
Removed sections: none

Deferred / TODO placeholders: none — all bracketed tokens resolved.

Templates requiring follow-up review (not modified by this command; read the
constitution at runtime):
  - .specify/templates/plan-template.md — verify "Constitution Check" gate
    references match principle names above (no placeholder drift observed).
  - .specify/templates/spec-template.md — no direct constitution references;
    no changes needed.
  - .specify/templates/tasks-template.md — API/handler tasks that produce or
    consume timestamps should apply the new format standard; no template edit
    required, enforcement happens at plan/implement time.
  - .claude/skills/speckit-*/SKILL.md and .github/skills/speckit-*/SKILL.md —
    generic workflow instructions, no hard-coded principle text found.
-->

# Go REST API Constitution

## Core Principles

### I. API Contract & RESTful Design
Every endpoint MUST be resource-oriented and follow RESTful conventions: nouns
in URL paths, HTTP methods mapped to CRUD semantics (GET/POST/PUT/PATCH/DELETE),
and status codes used per their standard meaning (2xx success, 4xx client
error, 5xx server error). All APIs MUST be versioned under a path prefix
(e.g., `/api/v1/...`); breaking changes require a new version, not an in-place
mutation of an existing one. Every endpoint MUST have its request/response
shape documented (OpenAPI/Swagger or equivalent) before or alongside
implementation, and responses MUST use a consistent JSON envelope (uniform
field casing, consistent error shape) across the service. Every timestamp
value in API input or output (request bodies, query params, response
fields) MUST be formatted as `YYYY-MM-DDTHH:mm:ss±H:mm` — date, time to the
second, and a UTC offset — for example `2026-09-02T14:30:00+7:00`; a
timestamp missing the offset, truncated to date-only, or expressed with
non-second precision is non-compliant.

Rationale: Consumers (internal services, frontends, third parties) depend on
predictable, stable contracts. Undocumented or inconsistent APIs create
integration risk that is expensive to unwind once clients exist. A single,
explicit timestamp format (including offset) removes ambiguity across
client time zones and prevents silent misinterpretation of date-only or
offset-less values.

### II. Test-First Development (NON-NEGOTIABLE)
TDD is mandatory for all handler, service, and repository code: tests are
written first, reviewed/approved, confirmed to fail, and only then is
implementation written (Red-Green-Refactor). Go's standard `testing` package
MUST be used with table-driven tests for business logic. Every HTTP handler
MUST have tests covering the success path and its documented error paths.
Every repository/DAO function that touches PostgreSQL MUST have an
integration test executed against a real (or ephemeral/dockerized) Postgres
instance — mocking the database driver for these tests is prohibited.

Rationale: A REST API's correctness is defined by its contract and its data
integrity guarantees; both are best protected by tests written before code,
and database behavior (constraints, transactions, query correctness) cannot
be trusted from mocks alone.

### III. Database Integrity & Migrations
PostgreSQL is the single system of record. All schema changes MUST be
expressed as versioned, ordered migration files (e.g., via `golang-migrate`
or equivalent) with both `up` and `down` scripts; manual/ad-hoc schema edits
against any environment are prohibited. Application code MUST access the
database only through a repository layer that isolates SQL from business
logic — handlers and services MUST NOT embed raw SQL. All queries MUST use
parameterized statements; string-concatenated SQL is prohibited. Multi-step
writes that must succeed or fail together MUST be wrapped in a database
transaction. Every table MUST include the standard audit columns
`created_at`, `created_by`, `updated_at`, `updated_by`, `deleted` (boolean),
and `deleted_at`; migrations that create a table without this column set are
non-compliant. Deletes MUST be implemented as soft deletes (setting
`deleted = true` and `deleted_at`) rather than physical `DELETE` statements,
and every read query MUST exclude soft-deleted rows (`deleted = false`)
unless it is explicitly querying the audit/history trail.

Rationale: Migrations make schema evolution reproducible and reversible
across environments. Isolating SQL and mandating parameterization keeps data
access testable and closes off SQL injection as an attack class by
construction rather than by review diligence.

### IV. Explicit Error Handling & Observability
Errors MUST be handled explicitly and wrapped with context (`fmt.Errorf` with
`%w`, or an equivalent typed error) as they cross layer boundaries; silent
failure, ignored error returns, and use of `panic` in the request-handling
path are prohibited except at the top-level recovery middleware. Every
service MUST emit structured (JSON) logs including a request/correlation ID
that is generated or propagated per request and returned to the caller in
error responses. Log lines MUST NOT contain secrets, credentials, or raw
PII. Any timestamp field in a log line MUST use the same format required
for API timestamps in Principle I (`YYYY-MM-DDTHH:mm:ss±H:mm`).

Rationale: In a service that other systems depend on, silent or ambiguous
failures are debugged in production under pressure. Structured logs and
correlation IDs are the minimum needed to trace a failure from a client
report back to the responsible code path and query.

### V. Simplicity & Standard-Library-First
Prefer the Go standard library (`net/http`, `database/sql`, `encoding/json`)
over third-party frameworks; a dependency MUST be justified by a concrete
need the standard library cannot reasonably meet (e.g., a router with path
parameters). New abstractions (interfaces, generic layers, plugin systems)
MUST NOT be introduced speculatively — only once at least two concrete
call sites need the abstraction. YAGNI applies: do not build for hypothetical
future scale, auth schemes, or backends that are not part of the current
feature's requirements.

Rationale: A small, standard-library-first codebase is easier to audit,
onboard into, and keep dependency-vulnerability-free — all of which matter
more for a REST API handling external traffic and a persistent data store
than premature flexibility does.

## Technology Stack & Security Requirements

- Language: Go, targeting the latest stable release; `go.mod` pins a minimum
  Go version and CI MUST build/test against that exact version.
- Database: PostgreSQL is the only supported relational datastore for this
  project; no alternate database engine may be introduced without a
  constitution amendment.
- Configuration & secrets (DB credentials, API keys, signing secrets) MUST be
  supplied via environment variables or a secret manager — never committed to
  the repository or hard-coded.
- All inbound data (request bodies, query params, path params) MUST be
  validated before use; validation failures return `4xx` with a machine-
  readable error code, never a raw internal error message.
- Dependencies MUST be kept current with security patches; `go.sum` is
  committed and dependency vulnerability scanning MUST run in CI.
- Transport MUST be TLS-terminated in every non-local environment.

## Development Workflow & Quality Gates

- Every change lands via pull request; at least one other reviewer MUST
  approve before merge into the main branch.
- CI MUST run `go vet`, a configured linter (e.g., `golangci-lint`), and
  `go test ./...` (including database integration tests against an ephemeral
  Postgres instance) on every PR; a red CI run blocks merge.
- Database migrations MUST be applied and verified in CI before a PR that
  depends on them can merge.
- Any deviation from a Core Principle (e.g., an unavoidable direct SQL query
  outside the repository layer, a new third-party framework) MUST be recorded
  with an explicit justification in the plan's Complexity Tracking section
  and approved during review — unjustified deviations are grounds for
  rejecting the PR.

## Governance

This constitution supersedes any conflicting team convention, template
default, or prior informal practice. All plans and PRs MUST verify
compliance with the Core Principles above; the plan template's Constitution
Check gate is the enforcement point before implementation begins.

**Amendment procedure**: Amendments are proposed via the `/speckit-constitution`
command (or direct edit reviewed as a PR), must state the rationale for the
change, and take effect once merged. Any amendment that redefines or removes
a principle MUST update every dependent template/command found to reference
the old principle text.

**Versioning policy**: This document follows semantic versioning:
- MAJOR: Backward-incompatible governance changes — a principle removed or
  redefined such that previously compliant work would no longer comply.
- MINOR: A new principle or materially expanded section added.
- PATCH: Clarifications, wording, or typo fixes with no semantic change.

**Compliance review**: Every `/speckit-plan` run MUST re-check its design
against these principles before proceeding to task generation; any
unresolved violation MUST be justified in Complexity Tracking or the plan
MUST be revised.

**Version**: 1.2.0 | **Ratified**: 2026-09-02 | **Last Amended**: 2026-09-02
