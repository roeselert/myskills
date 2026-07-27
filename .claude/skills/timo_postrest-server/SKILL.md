---
name: postgrest-server
description: Architecture and coding rules for PostgREST services on PostgreSQL — BCE layering mapped to schemas, business components (BC), exposed API schemas, views and RPC functions as the boundary, PL/pgSQL control logic, RLS-backed entities, role-based authorization, migrations, testing (pgTAP/integration/system), and project structure. Use when creating, generating, scaffolding, writing, or reviewing schemas, views, functions, policies, or endpoints in PostgREST projects. Not for ORM-backed application servers.
---

## Composition
- compose with `postgres-conventions` for all language-level Postgres rules (SQL/PL/pgSQL syntax, style, naming, privileges, functions, error handling, migrations, comments)
- compose with `bce` for the architectural pattern
- this skill specializes only the PostgREST context — it does not restate language-level rules

## Dependencies
- prefer capabilities in this order: core PostgreSQL, PostgREST features, supported extensions, external services
- do not introduce an application-tier service for logic the database can express; PostgREST exists so the schema *is* the API

## BCE Mapping

| BCE layer | PostgREST realization |
| --- | --- |
| boundary | exposed schema: views, RPC functions, `INSTEAD OF` triggers, grants, `COMMENT ON` |
| control | unexposed schema: PL/pgSQL and SQL functions holding procedural logic |
| entity | unexposed schema: tables, constraints, RLS policies, indexes |

- the database is the deployment unit; the database name reflects the project
- one BC = three schemas at most: `<bc>_boundary`, `<bc>_control`, `<bc>_entity`
- `db-schemas` lists only `*_boundary` schemas — control and entity schemas are never reachable over HTTP
- a BC need not have every layer; a control-only BC is valid for procedural shared concerns
- a BC boundary may be omitted only when nothing of that BC is exposed over HTTP
- `public` stays empty; no BC lives there
- file layout mirrors the schemas: `db/<bc>/{boundary,control,entity}/*.sql`, plus `db/migrations/`
- do not explain the BCE pattern in documentation

## Boundary Layer
- keep coarse-grained operations in the boundary — one request, one intent, one transaction
- PostgREST wraps each request in a single transaction; the boundary is therefore the transaction scope — never attempt transaction control in control or entity code
- expose read shapes as views, not tables; a table is never granted directly to a web-facing role
- expose writes as auto-updatable views for plain CRUD, and as RPC functions for anything with rules, multi-table effects, or a domain verb
- use `INSTEAD OF` triggers when a view must accept writes it cannot handle automatically
- boundary code contains no business logic — it maps request to a control call and result to a response shape
- name boundary views in plural after the resource (`orders`, `order_items`); name RPC functions after the domain verb (`submit_order`, not `order_post`)
- mark read-only RPC functions `STABLE` so they are reachable via `GET`; keep mutating functions `VOLATILE`
- put authorization decisions in the boundary: grants per exposed object, and role selection from the JWT
- `COMMENT ON` every exposed view, function, and non-obvious column — PostgREST publishes these as the OpenAPI description; this is the project's API documentation
- use a `db-pre-request` function for cross-cutting request setup (context GUCs, rate checks); keep it small and fast

## Control Layer
- implement procedural business logic in `<bc>_control`
- control functions are stateless and function-like; they take and return values, not request context
- control may be called by its own boundary or directly by another BC's boundary or control; the boundary is not a gate for cross-BC calls
- control never reads `request.*` settings — the boundary passes what is needed as parameters
- object names must not end with `_control`; the schema carries the layer

## Entity Layer
- maintain tables, constraints, and domain behavior in `<bc>_entity`
- entities are not anemic: put derivable values in generated columns and invariants in `CHECK`/`EXCLUDE` constraints rather than in calling code
- enable and `FORCE` RLS on every entity table; a table without a policy is unreachable, which is the correct default
- name policies after the rule they express (`orders_visible_to_owner`), and keep the policy in the same file as the table
- declare real foreign keys between entities — PostgREST derives resource embedding from them, so a missing FK silently removes API capability
- cross-BC entity references are allowed; excessive coupling is a refactoring signal — split, merge, or rebalance the BCs

## Roles & Authentication
- three role kinds: `authenticator` (`LOGIN NOINHERIT`, connects and switches roles), anonymous role (`db-anon-role`), and one role per API audience (e.g. `customer`, `staff`, `admin`)
- audience roles are `NOLOGIN`; the authenticator is `GRANT`ed each of them
- roles are named after audiences, not after BCs — a BC is not a security principal
- authorization is expressed as grants on boundary objects plus RLS policies keyed on the current role and JWT claims
- read claims via `current_setting('request.jwt.claims', true)::jsonb`; wrap the access in one `STABLE` helper function per project rather than repeating the expression
- never trust a claim that is not signed; do not accept identity from headers or query parameters

## Errors (HTTP mapping)
- raise errors with a SQLSTATE and let PostgREST map it — do not construct response bodies by hand
- rely on standard SQLSTATE mapping for the common cases (unique violation → 409, foreign key violation → 409, insufficient privilege → 401/403, no data → 404)
- force a specific status with the `PT<status>` SQLSTATE convention (e.g. `RAISE sqlstate 'PT402'`) when the standard mapping does not fit
- use the `PGRST` error-code form only when response headers or a custom body are genuinely required
- `MESSAGE` is the client-facing text, `DETAIL` is diagnostic, `HINT` is actionable — populate them deliberately
- prefer a constraint violation over a procedural check; a rejected write should fail at the entity, not at the boundary

## JSON
- let PostgREST serialize; do not hand-build JSON for standard responses
- shape responses with views and computed columns, not with `json_build_object` in the boundary
- use `jsonb` columns only for genuinely schemaless payloads; anything queried or constrained becomes a column
- return `jsonb` from RPC functions when the shape is dynamic; return a table type or `SETOF` when it is not
- accept a single `jsonb` parameter in RPC only for genuinely variadic input; otherwise use named typed parameters

## Migrations & Schema Cache
- all schema change goes through migrations under `db/migrations/`; nothing is applied by hand
- **always ask before changing the migration tool configuration or the PostgREST configuration file**
- on opening an existing project, load `AGENTS.md` (if present) before making changes
- do not create or change any files on opening an existing project; stop after initialization and wait for instructions
- do not generate schema initially in an empty project
- PostgREST caches the schema — after any DDL affecting exposed objects, reload with `NOTIFY pgrst, 'reload schema'`; add it as the last step of every migration touching a boundary schema
- the reload channel requires a direct (non-pooled) connection; issue it on the direct port

## Connections & Pooling
- PostgREST maintains its own pool (`db-pool`, `db-pool-acquisition-timeout`); size it against the database connection budget, not against expected traffic
- when PostgREST connects through a transaction-pooling pooler (PgBouncer, e.g. PlanetScale port `6432`): set `db-prepared-statements = false` and `db-channel-enabled = false` — prepared statements and `LISTEN`/`NOTIFY` do not survive transaction pooling
- reserve the direct port (`5432`) for migrations, schema-cache reloads, and admin work
- set `max-rows` so no endpoint can return an unbounded result set
- set `statement_timeout` on the audience roles; a web role must not be able to run an unbounded query

## Testing
- unit tests: pgTAP against control functions, constraints, and RLS policies; run in a rolled-back transaction; file suffix `_test.sql`
- integration tests: pgTAP against boundary views and RPC functions using `SET LOCAL ROLE`, verifying what each audience role can and cannot see; file suffix `_it.sql`
- system tests: HTTP against a running PostgREST instance, in a dedicated `st/` directory; use plain HTTP request files (hurl, `.http`) or a thin client — do not pull in a heavyweight test framework
- every exposed endpoint needs one system test for the happy path and one for the denied-authorization path
- run system tests after every change to a boundary schema
- in PoC mode (user-activated), system test execution is skipped
- at most three tests per object under test, per test kind

## Observability
- enable `pg_stat_statements`; treat `rows_read / rows_returned` and `total_time` as the primary optimization signals
- set `log_min_duration_statement` (OLTP: 1–3 s) and `log_lock_waits = on`
- set `application_name` per client in the pre-request function so slow queries are attributable to a caller
- enable PostgREST's `Server-Timing` output in non-production environments for per-request breakdown
- use OpenTelemetry at the reverse proxy for request-level tracing; do not build tracing into SQL

## README Guidelines
- write brief, 'to the point' README.md files for advanced developers
- use precise and concise language; avoid generic adjectives like "simple", "lightweight"
- do not include detailed project structure (file/folder listings); high-level BC descriptions are acceptable
- never list endpoints in the README — the OpenAPI output generated from `COMMENT ON` is the endpoint reference
- if BCs or modules are listed, provide links
- do not use "Orchestrates"; use a more specific verb
