---
name: postgres-conventions
description: Generic, composable PostgreSQL code conventions — modern SQL and PL/pgSQL syntax, schema and object naming, visibility and grants, functions, set-based query style, error handling, and documentation rules that apply across all Postgres contexts (migrations, PostgREST APIs, ETL jobs, application-owned schemas). Technology-neutral within the Postgres world; meant to be composed with context-specific skills (e.g. `postgrest-server`, `bce`). Use when writing, generating, or reviewing SQL, DDL, or PL/pgSQL anywhere the composed skill does not already specify style. Triggers on "Postgres conventions", "SQL style", "PL/pgSQL", "modern Postgres", "idiomatic SQL", or any request to write or review Postgres code where context-specific skills do not already cover style.
---

## Scope

- Language-level rules for PostgreSQL 17+ only — SQL and PL/pgSQL syntax, style, naming, visibility, functions, error handling, comments.
- Architecture (BCE, layering, schema partitioning by responsibility) is **not** in this skill — see `bce`.
- HTTP exposure, roles-as-API, JWT, deployment, and pooling rules are **not** in this skill — see `postgrest-server`.
- When a composed skill specifies a rule, the composed skill wins; this skill is the fallback baseline.

## Version & Syntax

- target PostgreSQL 17+; assume all features are GA — never depend on unreleased or patched-in behavior
- use modern syntax naturally: `GENERATED ALWAYS AS IDENTITY`, `MERGE`, `JSON_TABLE`, SQL/JSON constructors, `WITH ... AS MATERIALIZED`, `ANY`/`ALL` array predicates
- use `BIGINT GENERATED ALWAYS AS IDENTITY` for surrogate keys; never `SERIAL`, never random UUIDv4 as PK (use `uuidv7()` when a UUID is required)
- use `TIMESTAMPTZ`, never `TIMESTAMP` without time zone
- use `TEXT` with a `CHECK` constraint over `VARCHAR(n)`
- use standard SQL casts (`CAST(x AS bigint)`) in portable DDL; `x::bigint` is acceptable in internal code
- use `LANGUAGE sql` function bodies with the standard `BEGIN ATOMIC ... END` form where the logic is a single statement or expression — it is parsed and dependency-tracked at creation time
- use dollar quoting (`$func$`) only where `BEGIN ATOMIC` is not applicable (PL/pgSQL)

## Built-in Capabilities First

- use built-in Postgres capabilities over hand-written logic — constraints, generated columns, `ON CONFLICT`, window functions, `GROUP BY GROUPING SETS`, range types, `tstzrange` + exclusion constraints
- use the most specific type for the domain — `numeric` for money, `interval` over integer minutes, `inet`/`cidr` over text, `jsonb` over `json`, domains over repeated `CHECK` clauses
- enforce invariants in the database (`NOT NULL`, `CHECK`, `FOREIGN KEY`, `EXCLUDE`) rather than in calling code
- only use extensions verified as available on the target platform; check the provider's extension list before designing around one

## Visibility & Privileges

- schema membership, not object naming, expresses visibility — internal objects live in unexposed schemas
- `REVOKE ALL ... FROM PUBLIC` on every schema and function; grant explicitly and minimally
- grant on the smallest surface that satisfies the caller: `EXECUTE` on a function over `SELECT` on its underlying tables
- default to `SECURITY INVOKER` functions; reserve `SECURITY DEFINER` for the few cases that must legitimately cross a privilege boundary
- every `SECURITY DEFINER` function must set `SET search_path = ''` and fully qualify all object references
- enable and `FORCE` row level security on every table holding tenant- or user-scoped data
- never grant table-level `SELECT` to an externally reachable role when a view or function can express the same access

## Schemas, Tables & Functions

- one schema per responsibility; never a single monolithic `public` schema — leave `public` empty
- create a view or function only when it carries a name worth having; do not wrap a single table in a passthrough view with no added semantics
- use views for read shapes, functions for operations with parameters or side effects
- prefer a table with a foreign key over a native `ENUM` type for finite value sets — enum values cannot be removed or reordered and `ALTER TYPE ... ADD VALUE` has transactional restrictions; use a `CHECK (col IN (...))` constraint for small, stable sets
  - *deviation note:* this overrides the generic "model value objects as enums" rule from `bce`; Postgres enums are the wrong tool for evolvable domains
- mark every function's volatility explicitly (`IMMUTABLE`, `STABLE`, `VOLATILE`) and add `PARALLEL SAFE` where it holds — the planner relies on it
- mark functions `STRICT` when a null argument means a null result
- prefer composition of small functions over one long procedural body
- create additional objects only if they decrease complexity and increase readability

## Naming

- name schemas, tables, and functions after their responsibilities, not technical concerns
- singular `snake_case` for tables and columns (`user_account`, `order_item`, `created_at`)
- lowercase unquoted identifiers only — never quoted mixed-case (`"userName"`)
- avoid meaningless suffixes: `*_tbl`, `*_data`, `*_info`, `*_mgr`, `*_helper`
- reserve role-specific suffixes for objects that fulfill that role: `*_idx` for indexes, `*_check` / `*_fkey` / `*_pkey` for constraints, `*_v` only if views must be distinguishable
- avoid the `get_` prefix on functions; name the result — `active_orders(...)` not `get_active_orders(...)`
- verbs for functions with effects (`archive_order`), nouns for read functions and views (`order_summary`)
- one canonical naming pattern per project; never mix

## Queries & Set Orientation

- prefer set-based statements over `FOR ... LOOP` in PL/pgSQL — a loop over a result set is almost always a rewritable single statement
- never `SELECT *` in stored code; list columns explicitly
- always bound potentially unbounded reads with `LIMIT`
- keep predicates SARGable — no functions on indexed columns unless a matching expression index exists
- prefer `EXISTS` over `IN (subquery)`; prefer `JOIN` over correlated subqueries
- prefer `UNION ALL` over `UNION` unless deduplication is required
- prefer keyset (cursor) pagination over `OFFSET` for deep pages
- use CTEs for readability, not as optimization fences; add `MATERIALIZED` / `NOT MATERIALIZED` deliberately when it matters
- prefer a named CTE over deeply nested subqueries when readability suffers
- always index foreign key columns — Postgres does not create them automatically
- verify plans with `EXPLAIN (ANALYZE, BUFFERS)` before claiming an index is used

## Code Style

- KISS and YAGNI — implement the simplest thing that satisfies the constraint
- never over-engineer; ask before adding triggers, extension points, or generalized abstractions
- lowercase keywords and identifiers; one clause per line for multi-clause statements
- prefer multiple simpler statements to one dense expression
- prefer `format('...%I...%L', ident, literal)` for dynamic SQL — never string concatenation of user input
- avoid dynamic SQL entirely unless the object name is genuinely variable
- use named-notation arguments (`func(order_id => 42)`) when a call has more than two parameters or any defaults
- extract repeated literals into a `IMMUTABLE` constant function, a domain default, or a configuration table — define once, change once
- extract non-trivial calculations and complex boolean predicates into named functions or generated columns so call sites read as intent
- use guard clauses (early `RETURN`) over deeply nested `IF`/`ELSE`
- avoid triggers for business logic; reserve them for invariant enforcement, audit trails, and denormalization maintenance, and name them for what they enforce
- keep transactions short; never hold a transaction open across an external call
- use `ON CONFLICT` over read-then-write races
- avoid `NULL` semantics where a value is always required — `NOT NULL` on as many columns as possible
- add `created_at TIMESTAMPTZ NOT NULL DEFAULT now()` to all tables

## Error Handling

- raise with a specific `ERRCODE` — never bare `RAISE EXCEPTION 'something failed'` without a SQLSTATE
- use the standard SQLSTATE classes where one fits (`23505` unique violation, `22023` invalid parameter, `P0002` no data found); define project-specific codes in a documented, reserved range
- supply `MESSAGE`, `DETAIL`, and `HINT` deliberately — `MESSAGE` is user-facing, `DETAIL` is diagnostic, `HINT` is actionable
- do not catch and swallow: an `EXCEPTION` block must log with context or re-raise
- do not use `EXCEPTION` blocks for control flow — each one creates a subtransaction and costs performance
- prefer constraint violations over procedural validation; let the database reject bad data

## Logging & Observability

- use `RAISE LOG` / `RAISE NOTICE` with structured context; never write diagnostics into application tables as a substitute
- keep `pg_stat_statements` enabled for baseline query telemetry
- record intent-carrying application context via `SET LOCAL application_name` or session GUCs, not by embedding markers in SQL text

## Migrations

- migrations are forward-only, one logical change per file, ordered and immutable once applied
- **never edit a migration that has been applied to any shared environment** — write a new one
- every migration must be transactional unless the statement forbids it (`CREATE INDEX CONCURRENTLY`, `ALTER TYPE ... ADD VALUE`); isolate non-transactional statements into their own migration
- expand/contract for breaking changes: add new, backfill, switch reads, switch writes, drop old — never in one step
- always take an explicit `lock_timeout` and `statement_timeout` in DDL migrations to avoid queueing behind long readers
- **always confirm with a human before any destructive operation** — `DROP`, `TRUNCATE`, dropping indexes or partitions, resetting statistics
- run DDL on a direct connection, never through a transaction-pooling pooler

## Testing

- use pgTAP for database unit tests; run them inside a transaction that is rolled back
- test constraints, RLS policies, and function contracts — not generated columns, plain views, or things that cannot fail
- create minimalistic tests first; at most three tests per object under test
- test names describe the guaranteed behavior, not the mechanics; do not prefix with `test_` or `should_`
- a precise equality assertion makes weaker assertions on the same value obsolete

## Comments & Documentation

- default to no comments
- only comment when the *why* is non-obvious: a hidden constraint, a workaround for planner behavior, a deliberate denormalization
- do not explain *what* a statement does — well-named objects do that
- use `COMMENT ON` for the contract of externally visible objects (schemas, views, functions, columns); it is machine-readable and survives dumps
- do not write `COMMENT ON` text that restates the object signature

## Composition with Other Skills

- this skill defines only language-level Postgres conventions; architecture, HTTP exposure, authentication, build, and deployment come from the composed skill (e.g. `postgrest-server`, `bce`)
- when a composed skill adds or refines a rule, apply it on top of these rules; the composed skill always specializes, never contradicts
- if a composed skill is silent on a topic covered here, these rules apply by default
