---
name: npm-bce-layout
description: Structures client-only, offline-first PWAs with a local PGlite (Postgres-in-WASM) database using Maven/Gradle-style source roots - src/main plus src/test/unit and src/test/e2e, reverse-domain package folders such as de/roeselert/order - and Boundary-Control-Entity-View layering, implemented with modern Node idioms (ESM, subpath imports, node:test, node --run, esbuild). Use this skill whenever the user sets up a new npm, Node or PWA project, adds a feature or component to one, asks where a file belongs, wires up package.json scripts, configures node:test or coverage, sets up esbuild, a service worker, migrations, IndexedDB/OPFS persistence, or PGlite - and whenever they mention Spring Boot, Gradle or Maven layout, BCE, boundary/control/entity, offline-first, local-first, or "de.roeselert", even if they never name this convention explicitly. Also use when reviewing or refactoring an existing browser-side Node project toward this structure.
---

# Client-only PWA layout: BCE components, Node-native tooling

This skill applies Maven/Gradle source roots and Adam Bien's Boundary-Control-Entity pattern
to a **client-only PWA**, using plain Node as the whole toolchain. Target stack: **Node 22 LTS
or newer, ESM, `node:test`, esbuild, PGlite (`@electric-sql/pglite`) as the local Postgres**.

There is **no server process and no backend**. `dist/` is a folder of static files served by
any static host or CDN. All state lives in the browser, in a real Postgres compiled to WASM,
persisted to IndexedDB or OPFS. Node appears only in `build.js` and in the test runner.

The layering is Java-inspired; the *code* is not. Two rules keep it Node-native rather than
Java-in-JavaScript:

- **One module system, one extension**: `"type": "module"` plus `.js` everywhere. No `.mjs`,
  no `require`, no dual builds. `.mjs` is only worth it inside a CommonJS package, which this
  is not.
- **`src/main` is web code, not Node code.** It may use web-standard globals only. Node
  builtins live in `build.js` and in `src/test`. This is what lets the same module run
  unchanged in `node:test` and in the browser bundle, with no jsdom and no mocking layer.

| Free to use in `src/main` | Never in `src/main` |
|---|---|
| `fetch`, `URL`, `AbortController`, `structuredClone`, `crypto.randomUUID()`, `TextEncoder`, `Intl`, `queueMicrotask`, `Worker` | `node:*` imports, `process`, `Buffer`, `__dirname`, `require` |

## Directory layout

Always use this skeleton. `<component>` is a business capability (`order`, `billing`,
`customer`) - never a technical layer name (`utils`, `services`, `models`).

```
<project>/
├── package.json
├── .nvmrc                          # matches engines.node
├── .npmrc                          # engine-strict=true
├── build.js                        # the "Gradle build task"
├── src/
│   ├── main/
│   │   ├── de/roeselert/
│   │   │   ├── application.js      # composition root: opens DB, migrates, mounts components
│   │   │   ├── common/
│   │   │   │   ├── boundary/router.js
│   │   │   │   ├── control/cache-control.js
│   │   │   │   ├── view/app-update-button.js
│   │   │   │   └── entity/
│   │   │   │       ├── database.js        # opens the one PGlite instance
│   │   │   │       ├── database-worker.js # worker entry point (separate bundle)
│   │   │   │       └── migrator.js        # runs discovered migrations in order
│   │   │   └── <component>/
│   │   │       ├── boundary/       # exported facade + route registration
│   │   │       ├── control/        # business logic, orchestration
│   │   │       ├── entity/         # data shapes, validation, SQL, repositories, migrations/
│   │   │       └── view/           # custom elements / render functions, css
│   │   └── static/
│   │       ├── index.html
│   │       ├── manifest.webmanifest
│   │       ├── sw.js               # copied verbatim, never bundled
│   │       └── icons/              # icon.svg, 192.png, 512.png, maskable-512.png,
│   │                               # apple-touch-icon.png
│   └── test/
│       ├── unit/
│       │   └── de/roeselert/<component>/{boundary,control,entity}/
│       ├── e2e/
│       │   └── de/roeselert/<component>/
│       └── support/                # fixtures, DB + server helpers (the one non-mirrored folder)
└── dist/                           # build output, git-ignored, THE deployable
    ├── index.html  manifest.webmanifest  sw.js
    ├── app.js  db-worker.js  chunk-*.js
    └── vendor/pglite/              # copied wasm + data, not bundled
```

**File names are kebab-case**, like the rest of the npm and web ecosystem; exported class names
stay PascalCase (`order-repository.js` exports `OrderRepository`). View files need a hyphen for
custom elements anyway, so this keeps one rule instead of two. If an existing project already
uses PascalCase files, leave it - consistency beats migration.

The test trees **mirror the main tree** package-for-package, exactly like Maven. The test for
`src/main/de/roeselert/order/control/price-calculator.js` lives at
`src/test/unit/de/roeselert/order/control/price-calculator.test.js`. That mirroring makes
"where is the test for X" a mechanical question instead of a search.

## The four layers

| Layer | Contains | Depends on | Example |
|---|---|---|---|
| `boundary` | The component's exported facade, route registration, use-case entry points. No HTTP. | control, entity | `order-boundary.js` |
| `control` | Business rules, orchestration, pure functions | entity | `price-calculator.js` |
| `entity` | Data shapes, invariants, validation, **all SQL**, repositories, `migrations/` | the DB handle only | `order.js`, `order-repository.js` |
| `view` | Custom elements / render functions, css for this component | the component's boundary | `order-form.js`, `order.css` |

### Dependency rules

1. **Cross-component imports go through `boundary` only.** `billing/control/*` may import
   `order/boundary/order-boundary.js`, never `order/control/*` or `order/entity/*`.
2. **Within a component, dependencies point inward**: view → boundary → control → entity.
   Never the reverse.
3. **Only `entity` writes SQL.** `control` and `boundary` never see a query string and never
   touch the PGlite handle. This is the rule that survives replacing PGlite later.
4. **No cross-component SQL.** A join across two components' tables is the same violation as
   importing their entities; read through the other component's boundary instead. Give each
   component its own Postgres schema (`CREATE SCHEMA "order"`) so the violation fails loudly
   rather than silently working.
5. **`application.js` is the only place that knows all components.** It opens the database,
   runs migrations, then hands each boundary its dependencies.

Long-running work in `control` takes an `AbortSignal` rather than inventing a cancellation
protocol - it is a web standard, so it works identically in the browser and under `node:test`,
and `node:test` passes `t.signal` straight into it.

Truly shared code goes in `de/roeselert/common/` with the same BCE layering - but resist it.
Two components needing the same helper is usually a sign a third component is hiding there.

## Imports: subpath imports, not `../../../..`

Use the `imports` field in package.json. This is Node's native answer to path aliases - no
tsconfig `paths`, no bundler-only resolver, no `NODE_PATH`:

```json
{
  "imports": {
    "#de/*": "./src/main/de/*",
    "#test/*": "./src/test/support/*"
  }
}
```

```js
import { Order } from '#de/roeselert/order/entity/order.js';
import { freshDatabase } from '#test/database.js';
```

Node resolves this natively (so unit tests need zero config) **and** esbuild honours it when
bundling. Browsers do not resolve `#` specifiers - that is fine, they only ever see the bundle.
Include the `.js` extension: ESM does not do extension guessing, and omitting it is the single
most common ESM error. Set this up at init; adding it later means rewriting every import.

## The database

One PGlite instance per application, created in the composition root and injected downward.
Never `new PGlite()` inside a component.

**Always run PGlite in a web worker in the browser.** Two tabs holding the same IndexedDB or
OPFS database directly will lock or corrupt it; `PGliteWorker` elects one leader tab and
proxies the others, and OPFS access-handle persistence only works off the main thread at all.

```js
// common/entity/database-worker.js   - its own esbuild entry point
import { PGlite } from '@electric-sql/pglite';
import { worker } from '@electric-sql/pglite/worker';

worker({
  async init(options) {
    return new PGlite({ dataDir: options.dataDir ?? 'idb://app' });
  },
});
```

```js
// common/entity/database.js
import { PGliteWorker } from '@electric-sql/pglite/worker';

export function openDatabase({ dataDir = 'idb://app' } = {}) {
  return PGliteWorker.create(new Worker('/db-worker.js', { type: 'module' }), {
    meta: { dataDir },
  });
}
```

Start with `idb://`; it works everywhere. Move to `opfs-ahp://` only when profiling shows
IndexedDB is the bottleneck, and expect to check browser support.

### Migrations

Each component owns its migrations as **`.js` modules exporting SQL strings**, not `.sql`
files - a module imports identically in Node unit tests and in the browser bundle, whereas
`.sql` needs an esbuild loader that Node has no equivalent for.

```js
// order/entity/migrations/001-init.js
export const id = '001-order-init';
export const sql = `
  CREATE SCHEMA IF NOT EXISTS "order";
  CREATE TABLE "order".orders (
    id text PRIMARY KEY,
    total_cents integer NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
  );
`;
```

`migrator.js` records applied ids in `common.schema_migrations` inside a transaction and
applies the rest in sorted order. The component's boundary exports its migration list;
`application.js` concatenates them. Migrations are **forward-only** - you cannot ask users to
restore a backup, so never edit a shipped migration, add a new one.

## package.json

```json
{
  "name": "<project>",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22" },
  "packageManager": "npm@10.9.0",
  "imports": {
    "#de/*": "./src/main/de/*",
    "#test/*": "./src/test/support/*"
  },
  "scripts": {
    "build": "node build.js",
    "dev": "node build.js --watch",
    "test": "node --test --experimental-test-coverage \"src/test/unit/**/*.test.js\"",
    "test:watch": "node --test --watch \"src/test/unit/**/*.test.js\"",
    "e2e": "node --run build && node --test --test-concurrency=1 --enable-source-maps \"src/test/e2e/**/*.test.js\"",
    "check": "node --run test && node --run e2e"
  },
  "dependencies": {
    "@electric-sql/pglite": "^0.3.0"
  },
  "devDependencies": {
    "esbuild": "^0.25.0",
    "playwright": "^1.5.0"
  }
}
```

- `"private": true` on an app is not optional - it is the only thing standing between a stray
  `npm publish` and your app on the public registry.
- **No `start` script**: the artifact is static. `dev` is the esbuild watch server.
- **No `clean` script**: `build.js` already removes `dist/` on every run, so a separate
  cross-platform `rm -rf` shim is dead weight.
- **`node --run`** (Node 22+) instead of `npm run` inside scripts: it skips the npm bootstrap
  and is noticeably faster. It deliberately does *not* run `pre`/`post` hooks, so never rely on
  them; chain explicitly as above.
- Pass a **quoted glob**, not a bare directory. `node --test src/test/unit/` fails with
  `MODULE_NOT_FOUND` because Node treats a positional path as a file to execute; the glob form
  recurses, and Node expands it itself, so the quotes matter.
- `e2e` builds first: the suite drives the real `dist/`, not sources. `--test-concurrency=1`
  because the tests share a browser and a port; `--enable-source-maps` so failures point into
  source rather than the minified bundle.
- `check` mirrors Gradle's `check`: everything that must pass before merging. In CI, run
  `npm ci` (never `npm install`) and commit the lockfile.
- Add `--test-coverage-lines=80 --test-coverage-branches=80` to `test` once coverage is
  meaningful; those thresholds fail the run rather than just printing a table.

`.npmrc` gets `engine-strict=true` and `.nvmrc` gets the same major version as `engines.node`,
so a contributor on Node 18 gets a clear error instead of a confusing ESM stack trace.

## The build script

Two entry points (app shell, DB worker), PGlite left external and vendored, static assets
copied. Discovery-based: a new component folder needs no registry edit.

```js
import { context } from 'esbuild';
import { cpSync, mkdirSync, readFileSync, rmSync, writeFileSync } from 'node:fs';
import { fileURLToPath } from 'node:url';
import { join } from 'node:path';

const ROOT = import.meta.dirname;              // no __dirname in ESM
const MAIN = join(ROOT, 'src/main/de/roeselert');
const DIST = join(ROOT, 'dist');
const watch = process.argv.includes('--watch');
const BUILD_ID = watch ? 'dev' : new Date().toISOString().replace(/\D/g, '').slice(0, 14);

// resolve the package, do not guess node_modules paths - this survives workspaces and pnpm
const pgliteDist = fileURLToPath(new URL('./', import.meta.resolve('@electric-sql/pglite')));

rmSync(DIST, { recursive: true, force: true });
mkdirSync(DIST, { recursive: true });
cpSync(join(ROOT, 'src/main/static'), DIST, { recursive: true });
cpSync(pgliteDist, join(DIST, 'vendor/pglite'), { recursive: true });

// stamp the build id into the copied service worker - it is not bundled, so no `define`
const sw = join(DIST, 'sw.js');
writeFileSync(sw, readFileSync(sw, 'utf8').replaceAll('__BUILD_ID__', BUILD_ID));

const ctx = await context({                    // top-level await: ESM, no IIFE wrapper
  entryPoints: [
    { in: join(MAIN, 'application.js'), out: 'app' },
    { in: join(MAIN, 'common/entity/database-worker.js'), out: 'db-worker' },
  ],
  bundle: true,
  format: 'esm',
  splitting: true,
  target: 'es2022',
  minify: !watch,
  sourcemap: true,
  outdir: DIST,
  external: ['@electric-sql/pglite', '@electric-sql/pglite/worker'],
  loader: { '.css': 'css', '.svg': 'dataurl' },
});

if (watch) {
  await ctx.watch();
  const { host, port } = await ctx.serve({ servedir: DIST, fallback: join(DIST, 'index.html') });
  console.log(`dev http://${host}:${port}`);
} else {
  await ctx.rebuild();
  await ctx.dispose();
  console.log('built dist/');
}
```

Note the `node:` prefix on every builtin: it is unambiguous, cannot be shadowed by an npm
package called `path`, and makes "is this Node-only code?" greppable - which is how you enforce
the src/main rule above.

**Why PGlite is `external`.** It loads `postgres.wasm` and its data file via runtime URLs that
esbuild cannot rewrite into a bundle. Copying the package's `dist/` and resolving the bare
specifier in the browser with an import map is the reliable path. Confirm the emitted asset
names for your installed version before relying on them.

```html
<!-- src/main/static/index.html -->
<script type="importmap">
{
  "imports": {
    "@electric-sql/pglite": "/vendor/pglite/index.js",
    "@electric-sql/pglite/worker": "/vendor/pglite/worker/index.js"
  }
}
</script>
<script type="module" src="/app.js"></script>
```

## PWA shell

`src/main/static/` holds `index.html`, `manifest.webmanifest`, icons, and `sw.js`. Keep `sw.js`
**import-free** so it is copied verbatim and never bundled - module service workers are still
unevenly supported, so register it classic:

```js
navigator.serviceWorker?.register('/sw.js');
```

The service worker precaches the shell plus `/vendor/pglite/*` cache-first. The wasm is
multiple megabytes and immutable per version, so cache it aggressively and bump the cache name
on release. Since the database is local, "offline" is the normal case, not a degraded mode -
the app must boot with zero network.

Routing: prefer hash routing, or configure the host's 404 fallback to `index.html`. With a
static host you have no rewrite rules by default.

## Testing

**Unit tests get a real Postgres.** In Node, `new PGlite()` with no `dataDir` is in-memory and
starts in well under a second, so repository tests run against actual SQL with no mocks, no
Docker, no worker. `node:test` runs **each test file in its own process**, so per-file
isolation is free - lean on it instead of building a reset helper.

```js
// src/test/support/database.js
import { PGlite } from '@electric-sql/pglite';
import { migrate } from '#de/roeselert/common/entity/migrator.js';
import { migrations } from '#de/roeselert/order/boundary/order-boundary.js';

export async function freshDatabase(t) {
  const db = new PGlite();
  await migrate(db, migrations);
  t?.after(() => db.close());        // cleanup registered where it is created
  return db;
}
```

```js
// src/test/unit/de/roeselert/order/entity/order-repository.test.js
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';       // always /strict - never the loose variant
import { freshDatabase } from '#test/database.js';
import { save, findById } from '#de/roeselert/order/entity/order-repository.js';

describe('order repository', () => {
  it('round-trips an order', async (t) => {
    const db = await freshDatabase(t);
    await save(db, { id: 'o-1', totalCents: 1250 });
    assert.deepEqual(await findById(db, 'o-1'), { id: 'o-1', totalCents: 1250 });
  });

  it('rejects a negative total', async (t) => {
    const db = await freshDatabase(t);
    await assert.rejects(() => save(db, { id: 'o-2', totalCents: -1 }));
  });
});
```

- `entity` tests: real in-memory PGlite, assert SQL and constraints.
- `control` tests: pure functions, no database, no fakes needed if rule 3 held. If you find
  yourself reaching for `mock.method`, the dependency probably belongs in a parameter.
- `boundary` tests: the facade against a fresh database - the most valuable unit tests here.
- `view`: do not chase DOM coverage in unit tests. Keep render functions thin and cover them
  in e2e.
- Use `mock.timers.enable()` from `node:test` for anything time-based rather than sleeping;
  `--test-name-pattern` and `it.only` (with `--test-only`) narrow a failing run.

**E2E** drives the built `dist/` with Playwright from `node:test`: serve `dist/` with a small
`node:http` handler on port 0 (let the OS pick, so parallel checkouts never collide), then
`chromium.launch()` once in a `before`, one fresh `browser.newContext()` per test so IndexedDB
starts empty, and close both in `after`. Assert persistence explicitly by reloading the page
mid-test - that reload is the only place the real storage path gets exercised.

## Adding a component

```bash
node scripts/scaffold-component.js <component-name>
```

Read `scripts/scaffold-component.js` first if the project deviates from the default layout
(different reverse-domain than `de.roeselert`) - the constants at the top are meant to be
adjusted. It must also emit `entity/migrations/001-<component>-init.js` with the
`CREATE SCHEMA` statement.

By hand: create all four layer folders plus the mirrored `src/test/unit/...` folders, skipping
only layers that make no sense (a headless component has no `view/`). Empty layer folders
communicate intent and stop people inventing ad-hoc locations later.

## Applying this to an existing project

1. Add `"private": true`, `"type": "module"`, `engines`, and the `imports` map to package.json.
   Convert any leftover `require`/`module.exports` first - mixed module systems will produce
   errors that look like layout problems and are not.
2. Add `common/entity/{database,database-worker,migrator}.js` and get one PGlite instance
   opening from the composition root before moving any feature code.
3. Move code component by component, starting with the most isolated one; rewrite its imports
   to `#de/...` as you move it.
4. Push its SQL down into `entity/`, then capture the current schema as that component's
   `001-init.js`.
5. Strip `node:*`, `process`, and `Buffer` out of the moved code - that is what makes its tests
   runnable without jsdom.
6. Mirror its tests into `src/test/unit/de/roeselert/...`.
7. Repeat, then delete the old directories and update the scripts.

Do not attempt a big-bang move - the import rewrites and the SQL extraction are where mistakes
hide, and one component at a time keeps `node --run check` green throughout.

## Common failure modes

| Symptom | Cause |
|---|---|
| `ERR_MODULE_NOT_FOUND` on a relative import | Missing `.js` extension - ESM does not guess. |
| `MODULE_NOT_FOUND` from `npm test` | Unquoted glob or a bare directory passed to `node --test`. |
| `require is not defined` | CommonJS snippet pasted into an ESM project. |
| `__dirname is not defined` | Use `import.meta.dirname`. |
| Database locks or corrupt reads with two tabs open | PGlite opened on the main thread instead of via `PGliteWorker`. |
| Bundle 404s on `postgres.wasm` | PGlite was bundled instead of left `external` and vendored. |
| A unit test needs jsdom | Something in `src/main` reached for a Node or DOM-only API it should not have. |
