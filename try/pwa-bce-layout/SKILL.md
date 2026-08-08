---
name: npm-bce-layout
description: Structures client-only, offline-first PWAs with a local PGlite (Postgres-in-WASM) database using Spring Boot and Gradle conventions - src/main plus src/test/unit and src/test/e2e, reverse-domain package folders such as de.roeselert.order, and Boundary-Control-Entity-View layering. Use this skill whenever the user sets up a new npm, Node or PWA project, adds a feature or component to one, asks where a file belongs, wires up package.json scripts, configures node:test, sets up esbuild, a service worker, migrations, IndexedDB/OPFS persistence, or PGlite - and whenever they mention Spring Boot, Gradle or Maven layout, BCE, boundary/control/entity, offline-first, local-first, or "de.roeselert", even if they never name this convention explicitly. Also use when reviewing or refactoring an existing browser-side Node project toward this structure.
---

# Client-only PWA layout: Spring Boot conventions on PGlite

This skill applies the Maven/Gradle source layout and Adam Bien's Boundary-Control-Entity
pattern to a **client-only PWA**. Target stack: **Node 22+ as build/test tooling only, ESM,
`node:test`, esbuild, PGlite (`@electric-sql/pglite`) as the local Postgres**.

There is **no server process and no backend**. `dist/` is a folder of static files served by
any static host, CDN, or `file:`-adjacent dev server. All state lives in the browser, in a
real Postgres compiled to WASM, persisted to IndexedDB or OPFS. Node appears only in
`build.mjs` and in the test runner.

## Directory layout

Always use this exact skeleton. `<component>` is a business capability (`order`, `billing`,
`customer`) - never a technical layer name (`utils`, `services`, `models`).

```
<project>/
├── package.json
├── build.mjs                       # build script (the "Gradle build task")
├── src/
│   ├── main/
│   │   ├── de/roeselert/
│   │   │   ├── Application.mjs     # composition root: opens DB, migrates, mounts components
│   │   │   ├── common/
│   │   │   │   ├── boundary/Router.mjs
│   │   │   │   ├── control/CacheControl.mjs    # cache invalidation / update check
│   │   │   │   ├── view/app-update-button.mjs
│   │   │   │   └── entity/
│   │   │   │       ├── Database.mjs        # opens the one PGlite instance
│   │   │   │       ├── DatabaseWorker.mjs  # worker entry point (separate bundle)
│   │   │   │       └── Migrator.mjs        # runs discovered migrations in order
│   │   │   └── <component>/
│   │   │       ├── boundary/       # the component's exported facade + route registration
│   │   │       ├── control/        # business logic, orchestration
│   │   │       ├── entity/         # data shapes, validation, SQL, repositories, migrations/
│   │   │       └── view/           # DOM rendering / custom elements, css
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
│       └── support/                # fixtures, DB helpers (the one non-mirrored folder)
└── dist/                           # build output, git-ignored, THE deployable
    ├── index.html  manifest.webmanifest  sw.js
    ├── app.js  db-worker.js  chunk-*.js
    └── vendor/pglite/              # copied wasm + data, not bundled
```

The test trees **mirror the main tree** package-for-package, exactly like Maven. A test for
`src/main/de/roeselert/order/control/PriceCalculator.mjs` lives at
`src/test/unit/de/roeselert/order/control/PriceCalculator.test.mjs`. This mirroring is the
whole point - it makes "where is the test for X" a mechanical question instead of a search.

Use the `.mjs` extension throughout, or set `"type": "module"` in package.json and use `.js`.
Pick one and stay consistent; mixing them is the main source of ESM resolution pain.

## The four layers

| Layer | Contains | Depends on | Naming |
|---|---|---|---|
| `boundary` | The component's exported facade, route registration on the client router, use-case entry points. No HTTP. | control, entity | `OrderBoundary.mjs` |
| `control` | Business rules, orchestration, pure functions | entity | `PriceCalculator.mjs`, `OrderService.mjs` |
| `entity` | Data shapes, invariants, validation, **all SQL**, repositories, `migrations/` | the DB handle only | `Order.mjs`, `OrderRepository.mjs` |
| `view` | Custom elements / render functions, css for this component | the component's boundary | `order-form.mjs`, `order.css` |

Files use PascalCase when they export a class or a single cohesive module (`Order.mjs`),
matching Java conventions. View files use kebab-case (`order-form.mjs`) since custom element
definitions require a hyphen anyway.

### Dependency rules

1. **Cross-component imports go through `boundary` only.** `billing/control/*` may import
   `order/boundary/OrderBoundary.mjs`, never `order/control/*` or `order/entity/*`.
2. **Within a component, dependencies point inward**: view → boundary → control → entity.
   Never the reverse.
3. **Only `entity` writes SQL.** `control` and `boundary` never see a query string and never
   touch the PGlite handle. This is the rule that survives replacing PGlite later.
4. **No cross-component SQL.** A join across two components' tables is the same violation as
   importing their entities; read through the other component's boundary instead. Give each
   component its own Postgres schema (`CREATE SCHEMA "order"`) so the violation fails loudly
   rather than silently working.
5. **`Application.mjs` is the only place that knows all components.** It opens the database,
   runs migrations, then hands each boundary its dependencies.

Truly shared code goes in `de/roeselert/common/` with the same BCE layering - but resist it.
Two components needing the same helper is usually a sign a third component is hiding there.

## Imports: no `../../../..`

Use the `imports` field in package.json (subpath imports) for Java-style absolute imports:

```json
{ "imports": { "#de/*": "./src/main/de/*" } }
```

```js
import { Order } from '#de/roeselert/order/entity/Order.mjs';
```

This resolves natively in Node (so unit tests need no config) **and** esbuild resolves it when
bundling for the browser. Browsers do not resolve `#` specifiers themselves - that is fine,
they only ever see the bundle. Set this up when initializing the project; adding it later
means rewriting every import.

## The database

One PGlite instance per application, created in the composition root and injected downward.
Never `new PGlite()` inside a component.

**Always run PGlite in a web worker in the browser.** Two tabs holding the same IndexedDB or
OPFS database directly will lock or corrupt it; `PGliteWorker` elects one leader tab and
proxies the others, and OPFS access-handle persistence only works off the main thread at all.

```js
// common/entity/DatabaseWorker.mjs   - its own esbuild entry point
import { PGlite } from '@electric-sql/pglite';
import { worker } from '@electric-sql/pglite/worker';

worker({
  async init(options) {
    return new PGlite({ dataDir: options.dataDir ?? 'idb://app' });
  },
});
```

```js
// common/entity/Database.mjs
import { PGliteWorker } from '@electric-sql/pglite/worker';

export async function openDatabase({ dataDir = 'idb://app' } = {}) {
  return PGliteWorker.create(new Worker('/db-worker.js', { type: 'module' }), {
    meta: { dataDir },
  });
}
```

Start with `idb://`; it works everywhere. Move to `opfs-ahp://` only when profiling shows
IndexedDB is the bottleneck, and expect to check browser support.

### Migrations

Each component owns its migrations as **`.mjs` files exporting SQL strings**, not `.sql`
files - a `.mjs` module imports identically in Node unit tests and in the browser bundle,
whereas `.sql` needs an esbuild loader that Node has no equivalent for.

```js
// order/entity/migrations/001_init.mjs
export const id = '001_order_init';
export const sql = `
  CREATE SCHEMA IF NOT EXISTS "order";
  CREATE TABLE "order".orders (
    id text PRIMARY KEY,
    total_cents integer NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
  );
`;
```

`Migrator.mjs` records applied ids in `common.schema_migrations` inside a transaction and
applies the rest in sorted order. The component's boundary exports its migration list;
`Application.mjs` concatenates them. Migrations are **forward-only** - you cannot ask users to
restore a backup, so never edit a shipped migration, add a new one.

## package.json

```json
{
  "name": "<project>",
  "version": "0.1.0",
  "type": "module",
  "engines": { "node": ">=22" },
  "imports": { "#de/*": "./src/main/de/*" },
  "scripts": {
    "clean": "node -e \"require('node:fs').rmSync('dist',{recursive:true,force:true})\"",
    "build": "node build.mjs",
    "dev": "node build.mjs --watch",
    "test": "node --test \"src/test/unit/**/*.test.mjs\"",
    "e2e": "npm run build && node --test --test-concurrency=1 \"src/test/e2e/**/*.test.mjs\"",
    "check": "npm run test && npm run e2e"
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

- There is no `start` script - the artifact is static. `dev` is the esbuild watch server.
- Pass a **quoted glob**, not a bare directory. `node --test src/test/unit/` fails with
  `MODULE_NOT_FOUND` because Node treats a positional path as a file to execute; the glob
  form is what actually recurses, and Node expands it itself, so the quotes matter.
- `e2e` builds first: the e2e suite drives the real `dist/`, not sources.
- `--test-concurrency=1` for e2e because the tests share a browser and a port.
- `check` mirrors Gradle's `check`: everything that must pass before merging.
- `clean` uses `node -e` so it works on Windows. `node -e` runs as CommonJS even with
  `"type": "module"`, so `require` is correct there.

## The build script

Two entry points (app shell, DB worker), PGlite left external and vendored, static assets
copied. Discovery-based: a new component folder needs no registry edit.

```js
import { context } from 'esbuild';
import { cpSync, mkdirSync, readFileSync, rmSync, writeFileSync } from 'node:fs';

const MAIN = 'src/main/de/roeselert';
const DIST = 'dist';
const watch = process.argv.includes('--watch');
const BUILD_ID = watch ? 'dev' : new Date().toISOString().replace(/\D/g, '').slice(0, 14);

rmSync(DIST, { recursive: true, force: true });
mkdirSync(DIST, { recursive: true });
cpSync('src/main/static', DIST, { recursive: true });
cpSync('node_modules/@electric-sql/pglite/dist', `${DIST}/vendor/pglite`, { recursive: true });

// stamp the build id into the copied service worker - it is not bundled, so no `define`
const sw = `${DIST}/sw.js`;
writeFileSync(sw, readFileSync(sw, 'utf8').replaceAll('__BUILD_ID__', BUILD_ID));

const ctx = await context({
  entryPoints: [
    { in: `${MAIN}/Application.mjs`, out: 'app' },
    { in: `${MAIN}/common/entity/DatabaseWorker.mjs`, out: 'db-worker' },
  ],
  bundle: true,
  format: 'esm',
  splitting: true,
  target: 'es2022',
  minify: !watch,
  sourcemap: watch,
  outdir: DIST,
  external: ['@electric-sql/pglite', '@electric-sql/pglite/worker'],
  loader: { '.css': 'css', '.svg': 'dataurl' },
});

if (watch) {
  await ctx.watch();
  const { host, port } = await ctx.serve({ servedir: DIST, fallback: `${DIST}/index.html` });
  console.log(`dev http://${host}:${port}`);
} else {
  await ctx.rebuild();
  await ctx.dispose();
  console.log('built dist/');
}
```

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

`src/main/static/` holds `index.html`, `manifest.webmanifest`, icons, and `sw.mjs`. Keep
`sw.mjs` **import-free** so it is copied verbatim and never bundled - module service workers
are still unevenly supported, so register it classic:

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
Docker, no worker:

```js
// src/test/support/database.mjs
import { PGlite } from '@electric-sql/pglite';
import { migrate } from '#de/roeselert/common/entity/Migrator.mjs';
import { migrations } from '#de/roeselert/order/boundary/OrderBoundary.mjs';

export async function freshDatabase() {
  const db = new PGlite();
  await migrate(db, migrations);
  return db;
}
```

Every test gets its own instance - isolation is free, so never share one.

- `entity` tests: real in-memory PGlite, assert SQL and constraints.
- `control` tests: pure functions, no database, no fakes needed if rule 3 held.
- `boundary` tests: the facade against a fresh database - the most valuable unit tests here.
- `view`: do not chase DOM coverage in unit tests. Keep render functions thin and cover them
  in e2e.

**E2E** drives the built `dist/` with Playwright from `node:test`: launch a static server on
port 0, `chromium.launch()`, one fresh `browser.newContext()` per test so IndexedDB starts
empty. Assert persistence explicitly by reloading the page mid-test - that reload is the only
place the real storage path gets exercised.

## Adding a component

```bash
node scripts/scaffold-component.mjs <component-name>
```

Read `scripts/scaffold-component.mjs` first if the project deviates from the default layout
(different domain than `de.roeselert`, `.js` instead of `.mjs`) - the constants at the top are
meant to be adjusted. It must also emit `entity/migrations/001_<component>_init.mjs` and the
`CREATE SCHEMA` statement.

By hand: create all four layer folders plus the mirrored `src/test/unit/...` folders, skipping
only layers that make no sense (a headless component has no `view/`). Empty layer folders
communicate intent and stop people inventing ad-hoc locations later.

## Applying this to an existing project

1. Add `"type": "module"` and the `imports` map to package.json.
2. Add `common/entity/{Database,DatabaseWorker,Migrator}.mjs` and get one PGlite instance
   opening from the composition root before moving any feature code.
3. Move code component by component, starting with the most isolated one; rewrite its imports
   to `#de/...` as you move it.
4. Push its SQL down into `entity/`, then capture the current schema as that component's
   `001_init.mjs`.
5. Mirror its tests into `src/test/unit/de/roeselert/...`.
6. Repeat, then delete the old directories and update the scripts.

Do not attempt a big-bang move - the import rewrites and the SQL extraction are where mistakes
hide, and one component at a time keeps `npm run check` green throughout.
