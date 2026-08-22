---
name: npm-bce-layout
description: Structures PWA/Node.js/npm projects with Spring Boot's source layout (src/main plus src/test/unit and src/test/e2e) and Adam Bien's Boundary-Control-Entity-View layering, implemented with native Node idioms - subpath imports, factory-function wiring, node:test. Use this skill whenever the user sets up a new npm, Node or PWA project, adds a feature or component to one, asks where a file belongs, wires up package.json scripts, configures node:test, or mentions Spring Boot, Gradle or Maven layout, BCE, boundary/control/entity, or "de.roeselert" - even if they never name this convention explicitly. Also use when reviewing or refactoring an existing Node project toward this structure.
---

# npm project layout: BCE on idiomatic Node

Spring Boot's *source layout* and Adam Bien's *Boundary-Control-Entity* pattern, expressed in
plain ESM Node - not transplanted Java. Target stack: **Node 22+, ESM, `node:test`, esbuild,
zero runtime dependencies by default**. The project is a **monolith**: one Node process serves
the API and the bundled frontend assets.

The Java parts worth keeping are the ones that pay off: `src/main` / `src/test` separation, a
mirrored test tree, and components sliced by business capability with strict layer boundaries.
The Java parts worth dropping are the ones Node solves better: deep package folders, DI
containers, class-per-file ceremony.

## Directory layout

`<component>` is a business capability (`order`, `billing`, `customer`) - never a technical
layer name (`utils`, `services`, `models`).

```
<project>/
├── package.json
├── build.js                        # the "Gradle build task"
├── src/
│   ├── main/
│   │   ├── application.js          # composition root: wires boundaries, starts server
│   │   └── <component>/
│   │       ├── boundary/
│   │       │   ├── index.js        # the component's ONLY public export
│   │       │   └── order-routes.js
│   │       ├── control/            # business logic, orchestration
│   │       ├── entity/             # data shapes, invariants, repositories
│   │       └── view/               # index.html, css, client-side js
│   └── test/
│       ├── unit/<component>/{boundary,control,entity}/
│       └── e2e/<component>/
└── dist/                           # build output, git-ignored
    └── static/<component>/
```

The test trees **mirror the main tree** folder-for-folder. A test for
`src/main/order/control/price-calculator.js` lives at
`src/test/unit/order/control/price-calculator.test.js`. That mirroring is the point: "where is
the test for X" becomes mechanical instead of a search.

**Naming.** Files are kebab-case, the Node convention - `price-calculator.js`,
`order-repository.js`. PascalCase is for *exported class names*, not filenames. Test files end
in `.test.js`, which is what `node --test` recognises.

**Extensions.** Set `"type": "module"` and use `.js` everywhere. Only reach for `.mjs` if the
package must stay CommonJS for some other reason; mixing the two is the main source of ESM
resolution pain.

## The four layers

| Layer | Contains | May import | Example |
|---|---|---|---|
| `boundary` | HTTP routes, request/response mapping, error-to-status mapping, the component facade | control, entity | `order-routes.js`, `index.js` |
| `control` | Business rules, orchestration, pure functions | entity | `price-calculator.js` |
| `entity` | Data shapes, invariants, validation, repositories, domain errors | nothing in the component | `order.js`, `order-repository.js` |
| `view` | `index.html`, css, browser js | nothing server-side | `order-form.js` |

### Dependency rules

1. **Cross-component imports go through `boundary/index.js` only.** `billing/control/*` may
   import `#order`, never `#order/control/...`. The imports map below makes this the *only*
   thing that resolves, so the rule is enforced by Node rather than by discipline.
2. **Inside a component, dependencies point inward**: boundary → control → entity. An entity
   importing from control is a design smell.
3. **`entity` imports no framework and no I/O transport.** No HTTP, no `req`, no config
   lookups. That is what makes entities trivially unit-testable.
4. **`application.js` is the only file that knows all components.** It imports each boundary,
   registers routes, starts and stops the server.

Truly shared code goes in `src/main/common/` with the same layering - but resist it. Two
components needing the same helper usually means a third component is hiding there.

## Imports: subpath imports as the architecture guard

Java package folders in Node mean `../../../..`. Node's `imports` field (subpath imports) fixes
that *and* encodes the layering. Map each component to its facade, plus one wildcard for tests:

```json
{
  "imports": {
    "#order": "./src/main/order/boundary/index.js",
    "#billing": "./src/main/billing/boundary/index.js",
    "#common/*": "./src/main/common/*",
    "#main/*": "./src/main/*"
  }
}
```

- Production code imports other components as `import { orderBoundary } from '#order'`. Nothing
  else in `order/` is reachable - deep imports throw `ERR_PACKAGE_IMPORT_NOT_DEFINED`.
- **Inside** a component, use ordinary relative imports (`../control/price-calculator.js`). The
  tree is only two levels deep, so there is nothing to escape from.
- Tests use `#main/order/control/price-calculator.js` to reach internals directly.

Set this up at project init; adding it later means rewriting every import. Keys must start with
`#` - that is what marks them internal and keeps them out of the public `exports` surface.

Optional: swap implementations per environment with **import conditions** - useful for
replacing a repository with an in-memory double without a DI container.

```json
"#main/order/entity/order-repository.js": {
  "test": "./src/test/support/in-memory-order-repository.js",
  "default": "./src/main/order/entity/order-repository.js"
}
```

Activate with `node --test --conditions=test`. Use sparingly - explicit constructor injection
(below) is clearer for most cases.

## Wiring: factory functions, not a DI container

Spring's container becomes a plain composition root. Every layer exports a **factory that takes
its collaborators and returns closures**. No `new`, no decorators, no singleton globals - and
tests can inject fakes with an object literal.

```js
// src/main/order/entity/order-repository.js
export function createOrderRepository(db) {
  return {
    async findById(id) { /* ... */ },
    async save(order) { /* ... */ },
  };
}

// src/main/order/control/order-service.js
import { calculatePrice } from './price-calculator.js';

export function createOrderService({ orderRepository }) {
  return {
    async place(draft, { signal } = {}) {
      const order = { ...draft, total: calculatePrice(draft.items) };
      signal?.throwIfAborted();
      return orderRepository.save(order);
    },
  };
}

// src/main/order/boundary/index.js  <- the component's whole public API
export { createOrderBoundary } from './order-routes.js';
```

`application.js` is the only place that assembles them:

```js
import { createServer } from 'node:http';
import { createOrderBoundary } from '#order';
import { createOrderRepository } from '#main/order/entity/order-repository.js';

export function createApp({ db }) {
  const orderRepository = createOrderRepository(db);
  const order = createOrderBoundary({ orderRepository });
  return { routes: [...order.routes] };
}

export async function start({ port = process.env.PORT ?? 3000, db } = {}) {
  const server = createServer(dispatch(createApp({ db })));
  await new Promise((resolve) => server.listen(port, resolve));
  return server; // callers need .address().port and .close()
}
```

Export `createApp`/`start` and only call `start()` when run directly. That single line is what
lets e2e tests boot the real app in-process:

```js
if (import.meta.filename === process.argv[1]) start();
```

## Node-specific practices worth enforcing

- **`node:` prefix on every builtin** (`node:fs`, `node:http`). Unambiguous, and it fails loudly
  in a browser bundle instead of silently shimming.
- **Graceful shutdown in `application.js`.** Handle `SIGTERM`/`SIGINT`: stop accepting, then
  `server.close()`, then `server.closeIdleConnections()`. Set `process.exitCode` rather than
  calling `process.exit()`, which truncates pending writes.
- **Propagate `AbortSignal`** from boundary through control as an options argument. It is Node's
  native cancellation primitive; `signal.throwIfAborted()` beats bespoke timeout flags.
- **`AsyncLocalStorage` for request context** (`node:async_hooks`) - the Node answer to Spring's
  request scope. Store request id / user in the boundary, read it in logging. Do not use it to
  smuggle dependencies into control; that recreates a hidden container.
- **Domain errors live in `entity`**, carry a stable `code`, and use `{ cause }` when wrapping.
  Boundary owns the single `code → HTTP status` table. Control never sees a status code.
- **Validate at the boundary edge**, keep invariants in the entity. Handwritten guards or a JSON
  Schema validator both work; the layering matters more than the library.
- **`node:sqlite`** covers the repository's persistence needs in Node 22+ without a dependency.
  Keep the SQL inside `entity/*-repository.js` so nothing else knows it exists.
- **Config is read once, in `application.js`**, and passed down. `process.env` reads scattered
  through control make tests order-dependent.

## package.json

Script names mirror Gradle's tasks so the muscle memory transfers.

```json
{
  "name": "<project>",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22" },
  "imports": {
    "#order": "./src/main/order/boundary/index.js",
    "#common/*": "./src/main/common/*",
    "#main/*": "./src/main/*"
  },
  "scripts": {
    "clean": "node --eval \"require('node:fs').rmSync('dist',{recursive:true,force:true})\"",
    "build": "node build.js",
    "test": "node --test \"src/test/unit/**/*.test.js\"",
    "test:watch": "node --test --watch \"src/test/unit/**/*.test.js\"",
    "test:coverage": "node --test --experimental-test-coverage \"src/test/unit/**/*.test.js\"",
    "e2e": "node --test \"src/test/e2e/**/*.test.js\"",
    "check": "node --run test && node --run e2e",
    "start": "node src/main/application.js",
    "dev": "node --watch --env-file-if-exists=.env src/main/application.js"
  },
  "devDependencies": { "esbuild": "^0.25.0" }
}
```

- Pass a **quoted glob**, not a bare directory. `node --test src/test/unit/` fails with
  `MODULE_NOT_FOUND` - Node treats a positional path as a file to execute. Node expands the glob
  itself, so the quotes matter; an unquoted pattern gets mangled by the shell.
- Splitting unit and e2e is just two globs over the mirrored tree. No runner config file, no
  extra dependency - the main reason `node:test` suits this layout.
- `node --run` (Node 22+) executes a script without spawning npm. Use it inside composite
  scripts like `check`; it is noticeably faster and skips npm's lifecycle hooks.
- `--env-file-if-exists` loads `.env` natively - no `dotenv`.
- `clean` uses `node --eval` rather than `rm -rf` so it works on Windows. `--eval` runs as
  CommonJS even with `"type": "module"`, so `require` is correct there.

## Testing with node:test

Unit tests exercise one layer with fakes for the rest - the payoff for factory wiring:

```js
import { describe, it, mock } from 'node:test';
import assert from 'node:assert/strict';
import { createOrderService } from '#main/order/control/order-service.js';

describe('order service', () => {
  it('totals the line items before saving', async () => {
    const save = mock.fn(async (order) => order);
    const service = createOrderService({ orderRepository: { save } });

    await service.place({ items: [{ price: 5, qty: 2 }] });

    assert.equal(save.mock.calls[0].arguments[0].total, 10);
  });
});
```

Use `assert/strict` (never bare `assert`), `mock.timers` for anything clock-driven, and
`t.assert.snapshot()` with `--test-update-snapshots` for wide response objects.

E2E tests boot the real server on **port 0** so the OS picks a free port - the suite never
collides with a running dev server, and tests stay parallel-safe:

```js
import { after, before, it } from 'node:test';
import assert from 'node:assert/strict';
import { start } from '#main/application.js';

let baseUrl;
before(async () => {
  const server = await start({ port: 0 });
  baseUrl = `http://localhost:${server.address().port}`;
  after(() => server.close());
});

it('creates an order', async () => {
  const res = await fetch(`${baseUrl}/api/orders`, { method: 'POST', body: '{}' });
  assert.equal(res.status, 201);
});
```

Add `--test-concurrency=1` to the `e2e` script only if the suite shares a database or a browser.
Port 0 alone removes the usual reason for serialising.

## The build script

`build.js` is the Gradle `build` task: bundle each component's view into
`dist/static/<component>/`. Server code is plain ESM and needs no compilation.

```js
import { build } from 'esbuild';
import { readdirSync, existsSync, cpSync, mkdirSync } from 'node:fs';
import { join } from 'node:path';

const MAIN = 'src/main';
const OUT = 'dist/static';

const components = readdirSync(MAIN, { withFileTypes: true })
  .filter((d) => d.isDirectory() && existsSync(join(MAIN, d.name, 'view')))
  .map((d) => d.name);

for (const component of components) {
  const viewDir = join(MAIN, component, 'view');
  const outDir = join(OUT, component);
  mkdirSync(outDir, { recursive: true });

  const entry = join(viewDir, `${component}-view.js`);
  if (existsSync(entry)) {
    await build({
      entryPoints: [entry],
      bundle: true,
      minify: true,
      sourcemap: true,
      format: 'esm',
      target: 'es2022',
      outfile: join(outDir, 'bundle.js'),
    });
  }
  cpSync(join(viewDir, 'index.html'), join(outDir, 'index.html'));
}

console.log(`Bundled ${components.length} component view(s)`);
```

Discovery-based by convention: drop in a component folder with a `view/` directory and the build
picks it up - no central registry. In development, serve `src/main/<component>/view` unbundled so
the edit-reload loop stays instant; `application.js` picks the directory from `NODE_ENV`.

## Adding a component

```bash
node scripts/scaffold-component.mjs <component-name>
```

Read `scripts/scaffold-component.mjs` first if the project deviates from the default layout - the
constants at the top (root package folder, file extension) are meant to be adjusted. The
scaffolder also appends the new `#<component>` key to the imports map, which is the step most
often forgotten when creating by hand.

By hand: create all four layer folders (skip `view/` for a component with no UI), a
`boundary/index.js` that exports only the facade, the mirrored `src/test/unit/...` folders, and
the imports-map entry. Empty layer folders communicate intent and stop people inventing ad-hoc
locations later.

## Keeping the reverse-domain package

The `de/roeselert/` nesting is optional here - `#order` already namespaces the component, and npm
scopes (`@roeselert/<project>`) handle publication identity. Keep it only if it is a house
convention. If so, put it under `src/main/de/roeselert/` and change only the right-hand side of
the imports map:

```json
"#order": "./src/main/de/roeselert/order/boundary/index.js",
"#main/*": "./src/main/de/roeselert/*"
```

Every import statement, the test tree, and `build.js`'s `MAIN` constant are the only other things
that need to agree. Because imports are absolute, nothing else in the codebase notices the depth.

## Worked example

For a complete `order` component - routes, service, entity, view, and both test types written out
in full - read `references/example-component.md`. Consult it when the user wants to see how the
layers actually talk to each other, or when generating a first component in a fresh project.

## Migrating an existing project

Move in this order so the project keeps running at every step:

1. Add `"type": "module"` and an empty `imports` map to package.json.
2. Create `src/main/` and move the **most isolated component first**, whole.
3. Add its `#<component>` entry, rewrite its imports, and rename files to kebab-case as you move.
4. Mirror its tests into `src/test/unit/<component>/...`.
5. Repeat per component, then delete the old directories and update the scripts.

Do not attempt a big-bang move. The import rewrites are where mistakes hide, and one component at
a time keeps `node --run check` green throughout.
