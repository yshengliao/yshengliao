# ai*js — micro-runtime ecosystem

> AI-readable, domain-neutral TypeScript micro-runtimes. Pure-function cores, opt-in subpaths, zero cross-package dependencies. Built for browser, Node, and Web Worker; usable far beyond the games it started in.

| Package | Version | Role | dist (raw) | Repo |
|---|---|---|---|---|
| [aifsmjs](aifsmjs/) | 0.2.1 | Strict finite-state machine — pure `step()` lifecycle, opt-in effects, PBT adapter | 520 KB | [aifsmjs/README.md](aifsmjs/README.md) |
| [aiecsjs](aiecsjs/) | 0.2.1 | Archetype ECS — SoA TypedArray storage, SAB-ready snapshot transport | 960 KB | [aiecsjs/README.md](aiecsjs/README.md) |
| [aibridgejs](aibridgejs/) | 0.2.1 | Cross-context bridge — iframe / Flutter / mock, JSON envelope, `AbortSignal`-aware | 272 KB | [aibridgejs/README.md](aibridgejs/README.md) |

v0.3 cycle adds four siblings, all **published to npm at v0.1.1 with SLSA Level 3 provenance**:
[aieventjs](https://www.npmjs.com/package/aieventjs), [aipooljs](https://www.npmjs.com/package/aipooljs), [aiquadtreejs](https://www.npmjs.com/package/aiquadtreejs), [aiaudiojs](https://www.npmjs.com/package/aiaudiojs). See [§4](#4-v030-candidates) for sizes + strategy.

Companion: [aijs-integration-smoke](aijs-integration-smoke/) verifies all three coexist in a single TypeScript module under `tsc --noEmit --strict` with zero identifier collisions.

Retrospectives per release cycle live in [LEARNINGS.md](LEARNINGS.md) — read it for the "why" behind the conventions below, and for specific bug patterns / tooling traps encountered in past releases.

---

## 1. Ecosystem positioning

The three packages address three orthogonal concerns of stateful TypeScript apps. They **do not depend on each other** — each can be adopted in isolation, and the version numbers march together only for the convenience of a single audit-and-bump cadence.

| You need to model… | Reach for | Why |
|---|---|---|
| A flow of named states with guarded transitions, side effects, replay | `aifsmjs` | Pure `step(def, snap, evt, impl)` core. Definitions are plain data (JSON-serialisable). Effects are descriptors, not callbacks. Built-in `fast-check` PBT adapter. |
| A simulation, dependency graph, or system with thousands of homogeneous entities | `aiecsjs` | Archetype tables + bitmask queries, TypedArray columns, SAB-ready snapshot transport for Web Workers. Functional API; `pipe()` for system composition. |
| Two execution contexts that need a typed RPC channel (iframe, Flutter InAppWebView, dedicated worker, off-thread renderer) | `aibridgejs` | Transport-agnostic core + per-host adapter. Strict origin / source validation. `AbortSignal` end-to-end. Generic `call<T>()` for return-type assertions. |

### Integration shapes (compose by convention, not by dependency)

```ts
//  Web page  →  aifsmjs runtime drives scene flow
//                ↓ emit('scene', state)
//  aibridgejs.Bridge  →  postMessage  →  iframe widget
//                ↑ on('command')
//  aiecsjs.World     →  toJSON(world)  →  payload for bridge.emit
```

- **aifsmjs + aiecsjs**: an FSM owns the high-level scene/level state; the ECS owns frame-by-frame entity data. The FSM emits "enter level N" → app spawns ECS entities; ECS emits "objective met" → app sends event to FSM.
- **aiecsjs + aibridgejs**: stream a world snapshot to a sandboxed iframe for visualisation. Always run `toJSON(world)` first — the bridge's JSON envelope drops `Date`, `Map`, `Set`, and class prototypes (see [aiecsjs · Integration with aibridgejs](aiecsjs/README.md#integration-with-aibridgejs)).
- **aifsmjs + aibridgejs**: the FSM owns local UI state; `aibridgejs.call<T>(...)` is the side-effect channel to a native host (Flutter shell, parent frame, service worker).

None of these compositions add a `dependencies` entry. The packages remain independently publishable.

---

## 2. New-package conventions (template + rules)

Use this section as the checklist when starting a fourth `ai*js` package.

### 2.1 Naming contract

| Surface | Convention | Rationale |
|---|---|---|
| Factory | `createX(opts?)`, never a `class` constructor | Functional surface; easy to pass around, mock, and tree-shake. |
| Disposal | `dispose()` — **idempotent**; after dispose all public methods throw `XDisposedError` synchronously | Consistent teardown across the family. `aiecsjs` keeps `destroyWorld` as a deprecated alias for legacy ECS callers. |
| Reset | `reset()` — wipe back to initial state without releasing internal buffers | HMR-friendly; different concept than dispose. |
| Subscription | `on(event, handler, opts?: { signal?, once? }) => () => void` | Returns unsubscribe; honours `AbortSignal`. |
| Long-running / async | `(args, { signal?: AbortSignal })` | Every long-lived operation accepts a signal. |
| Domain neutrality | No `game`, `player`, `level`, `score`, `sprite`, `collision`, `physics`, `reel`, `spin`, `slot`, etc. in source or public docs | Game terminology belongs in usage examples, not the API surface. |

### 2.2 `package.json` shape

```jsonc
{
  "name": "aiXXXjs",
  "version": "0.1.0",
  "type": "module",
  "sideEffects": false,
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".":          { "types": "./dist/index.d.ts",     "import": "./dist/index.js",     "require": "./dist/index.cjs"     },
    "./feature":  { "types": "./dist/feature.d.ts",   "import": "./dist/feature.js",   "require": "./dist/feature.cjs"   }
  },
  "files": ["dist", "README.md", "README_ZHTW.md", "LICENSE", "llms.txt", "llms-full.txt"],
  "engines": { "node": ">=18.0.0" },
  "license": "MIT"
}
```

Rules that travel with this template:

- Dual ESM + CJS output. `tsup` is the build tool of choice (matches `aifsmjs` and `aibridgejs`).
- `sideEffects: false` so bundlers can drop unused subpaths.
- Subpath exports per optional capability. The root keeps the stable core; subpaths host opt-in extras (adapters, PBT helpers, persistence).
- **No `dependencies`**. Runtime is zero-dep. The only exceptions allowed are `peerDependencies` set to *optional* — see `aifsmjs`'s `fast-check`. Anything that needs more belongs in user-land.
- **No cross-package imports.** `aiXXXjs` must not `import 'aiYYYjs'`. Integration is a documentation concern, not a dependency.

### 2.3 Documentation contract (AI-readable)

Every package ships:

| File | Purpose |
|---|---|
| `README.md` (English, canonical) | Quick start, mental model, API reference, comparison table, FAQ. |
| `README_ZHTW.md` | Traditional-Chinese mirror — full parity. |
| `CHANGELOG.md` (+ `_ZHTW`) | Keep-a-Changelog + SemVer. |
| `STABILITY.md` (+ `_ZHTW`) | Per-export stability tag (`stable` / `experimental` / `internal` / `deprecated`) and `since` version. |
| `llms.txt` | [llmstxt.org](https://llmstxt.org/) discovery index. |
| `llms-full.txt` (+ `_ZHTW`) | Single-file complete API reference for LLM consumption. Generated by `scripts/build-llms-full.mjs`. |
| `api.json` | Machine-readable export manifest. Each entry carries `stability`, `since`, signature, and example. |
| `CONTRIBUTING.md` | Quick start, what gets in easily, what needs discussion. |
| `LICENSE` | MIT. |

Stability tags travel with the same vocabulary across packages: **stable**, **experimental**, **internal**, **deprecated**. Same tag, same expectation everywhere.

### 2.4 CI and quality gates

`prepublishOnly` runs in this order:

```bash
typecheck → lint → coverage → build → verify:exports → check:size
```

| Gate | Tool | Threshold |
|---|---|---|
| Typecheck | `tsc --noEmit` with `strict + noUncheckedIndexedAccess + exactOptionalPropertyTypes` | Zero errors. |
| Lint | [Biome](https://biomejs.dev/) (`biome.json` provided in every package) | Zero errors. `noExplicitAny` is `warn`. |
| Tests | Vitest behavioural tests | ≥95% statements / lines / functions / ≥90% branches. `aifsmjs` and `aibridgejs` enforce 100/100/100/≥90. |
| Build | `tsup` dual ESM/CJS + `.d.ts` | — |
| Exports | `scripts/verify-exports.mjs` | Every `package.json#exports` entry maps to an existing `dist/` file. |
| Size | `scripts/check-size.mjs` or `size-limit` | Per-subpath gzip budget; failure blocks publish. |

Publishing uses **npm OIDC trusted publisher** with provenance attestation; no long-lived `NPM_TOKEN`. `aifsmjs` and `aibridgejs` already ship this way; `aiecsjs` is wired up the same on 0.2+.

### 2.5 Package skeleton

```
aiXXXjs/
├── src/
│   ├── index.ts                 # root re-exports
│   └── <feature>/index.ts       # subpath entries
├── test/                        # vitest behavioural tests
├── examples/<NN>-<scenario>/    # tsx executable demos
├── scripts/
│   ├── verify-exports.mjs       # gate: package.json#exports vs dist/
│   ├── check-size.mjs           # gate: per-subpath gzip budget
│   └── build-llms-full.mjs      # generator: README + src → llms-full.txt
├── README.md / README_ZHTW.md
├── CHANGELOG.md / CHANGELOG_ZHTW.md
├── STABILITY.md / STABILITY_ZHTW.md
├── CONTRIBUTING.md
├── llms.txt / llms-full.txt
├── api.json
├── biome.json
├── tsup.config.ts
├── tsconfig.json / tsconfig.test.json
├── vitest.config.ts
├── LICENSE  (MIT)
└── package.json
```

When in doubt, copy [aifsmjs/](aifsmjs/) — it is the most-conformant member of the family.

---

## 3. v0.2.0 roadmap (delivered)

This section freezes what landed in the simultaneous 0.2.0 release across the three packages. Non-breaking unless flagged.

### aibridgejs 0.2.0
- **API**: `Bridge.call` is now generic — `call<T = unknown>(method, payload?, options?): Promise<T>`. Existing `as`-cast call sites should migrate.
- **Docs**: new "Error semantics (retry table)" section across English / Traditional-Chinese READMEs + `llms-full.txt`. Each of the five error classes is classified retryable / not retryable with a sample `callWithReset()` helper.

### aifsmjs 0.2.0
- **API**: `defineMachine` + `evalGuard` now throw a new `AsyncGuardError` on declared-`async` guards OR Promise-returning guards. Closes the gap where a cast slipped an async guard through and silently passed every transition. Exports: `AsyncGuardError`, `isAsyncGuardFn`.
- **Positioning**: README "Primary audience" rewritten — leads with stateful web flows (multi-step forms, checkout funnels, auth flows, tutorials, document workflows) and frames games as one application of the same pattern.
- **Examples**: two new non-game demos — `03-checkout-funnel/` (e-commerce funnel with payment + analytics effects + replay) and `04-form-wizard/` (multi-step form with back/next/jump + draft persistence).

### aiecsjs 0.2.0
- **API alias**: `disposeWorld` added as a non-breaking alias of `destroyWorld`, aligning with the ai*js ecosystem `dispose()` convention. `destroyWorld` is now `deprecated` (slated for removal in 1.0).
- **API extension**: every observer (`onAdd` / `onRemove` / `onSet` / `observe`) accepts `{ signal?: AbortSignal }`. New exported type `ObserverOptions`. Closes the long-standing observer cleanup gap noted in the AI audit.
- **Stability honesty**: `getEntityGeneration` and `packEntity` re-classified `stable → experimental`. Both return `0` / identity in 0.x; real values arrive with ABA-safe `EntityRef` in a later 0.x.
- **Docs**: `onSet` now carries explicit JSDoc + README clarification — it is a **low-level mutation hook**, not a reactive value-predicate query.
- **Tooling parity**: added `biome.json`, `scripts/verify-exports.mjs`, `CONTRIBUTING.md`, and `lint` / `format` / `verify:exports` scripts. Now matches `aifsmjs` and `aibridgejs`'s gate.

### Cross-package
- All three packages bumped to **0.2.0**.
- All CHANGELOGs updated (English + Traditional Chinese where applicable).
- 394 tests passing across the family (`aibridgejs` 105 + `aifsmjs` 142 + `aiecsjs` 147).

---

## 4. v0.3.0 candidates

The next cycle adds four small siblings, motivated by upcoming Svelte 5 + PixiJS game work and SvelteKit / Nuxt web apps. **All four are now published to npm with SLSA Level 3 provenance attestation** (GitHub Actions OIDC trusted publisher). See [LEARNINGS.md v0.3.0 cycle](LEARNINGS.md#v030-cycle--預備區) for the underlying mitt / Howler.js evaluation and the subagent + Codex pipeline retrospective.

| # | Package | Status | Actual gzip / budget | Strategy |
|---|---|---|---|---|
| 1 | [aieventjs](https://www.npmjs.com/package/aieventjs) | published — [v0.1.1 on npm](https://www.npmjs.com/package/aieventjs/v/0.1.1) (provenance) | 754 B / 800 B | Self-built — `mitt` is MIT-fork-friendly but is 35 lines of pure logic and stalled since 2023-07; cheaper to write from scratch with `unsubscribe`-return + `AbortSignal` + `dispose()` + wildcard `*` |
| 2 | [aipooljs](https://www.npmjs.com/package/aipooljs) | published — [v0.1.1 on npm](https://www.npmjs.com/package/aipooljs/v/0.1.1) (provenance) | 557 B / 700 B | Self-built — fixed-size object pool for PixiJS sprites / bullets / particles / DOM recyclers |
| 3 | [aiquadtreejs](https://www.npmjs.com/package/aiquadtreejs) | published — [v0.1.1 on npm](https://www.npmjs.com/package/aiquadtreejs/v/0.1.1) (provenance) | 967 B / 2000 B | Self-built — 2D quadtree for per-frame rebuild collision broadphase |
| 4 | [aiaudiojs](https://www.npmjs.com/package/aiaudiojs) | published — [v0.1.1 on npm](https://www.npmjs.com/package/aiaudiojs/v/0.1.1) (provenance) | 1423 B / 2000 B | Self-built shell — Howler.js is MIT, 9.7 KB gzip, mature; iOS unlock edge cases are WebKit-bound and unrewarding to rewrite. Shell delivers ai\*js conventions (`dispose()` / AbortSignal / first-class `crossfade()`) + `sound.nativeHowl` escape hatch. **Howler is a required `peerDependency ^2.2.4`** |

The same naming, lint, build, verify, and publish gates from [§2](#2-new-package-conventions-template--rules) apply unchanged. The same security baseline from [§5](#5-security-and-dependency-baseline) applies unchanged.

### Review pattern for 0.1.0 implementation

Each package's 0.0.1 → 0.1.0 implementation jump runs the three-parallel review pattern proven in v0.2.0 (see [LEARNINGS.md](LEARNINGS.md)):

- `/code-review` — correctness first opinion
- `/codex:review` — independent second opinion (caught regressions Claude introduced in v0.2.0 round-2)
- `/security-review` — **mandatory** for `aiaudiojs` (AudioContext lifecycle, postMessage in Howler internals, untrusted URL passed to `new Howl({ src })`); judgment call for the other three based on Round-1 findings.

Publish workflow for each package currently triggers only on `workflow_dispatch` (not on tag push) to prevent a 0.0.x stub from being released to npm by accident. Before the 0.1.0 publish, restore the `on: push: tags: ["v*"]` trigger and configure the npm Trusted Publisher entry.

---

## 5. Security and dependency baseline

Every package in this family must satisfy the following on every release. The list is enforced by code review, not by CI — most checks are hard to mechanise, but the violations are easy to spot.

### 5.1 Code

- **No `eval`, `new Function`, or other dynamic-code execution.** CSP-friendly; the family ships into iframes, Flutter WebViews, and CSP-restricted hosts.
- **No `Function.prototype.constructor` lookups on untrusted input.**
- **No hardcoded secrets.** Even test fixtures use environment variables.
- **Boundary validation.** Every input from outside the package (postMessage payloads, `JSON.parse`, `deserializeWorld`, snapshot adoption) is structurally validated. Magic + version headers on binary formats (`aiecsjs/serialize` uses `0x41494543 "AIEC"`).
- **Prototype-pollution awareness.** No `Object.assign({}, untrustedJson)` patterns. Prefer explicit field extraction or `structuredClone`.
- **Strict origin/source validation in transports.** `aibridgejs` iframe adapter rejects `targetOrigin: '*'` synchronously at construction and validates both `event.origin` and `event.source` on every inbound message.

### 5.2 Dependency policy

- **Zero runtime dependencies by default.** `aiecsjs` and `aibridgejs` are at zero. `aifsmjs` keeps `fast-check` only as an *optional* `peerDependency` so the PBT adapter can be tree-shaken when unused.
- **devDependencies travel with Dependabot.** Transitive advisories (e.g. esbuild CORS, vite path traversal) get patched promptly and published in a Security entry of the CHANGELOG — see [aibridgejs CHANGELOG 0.1.2](aibridgejs/CHANGELOG.md).
- **No cross-package imports at runtime.** `aifsmjs` ≠ `aiecsjs` ≠ `aibridgejs`; users compose them by convention.

### 5.3 Supply chain

- **npm OIDC trusted publisher** with provenance attestation (`--provenance`). No long-lived `NPM_TOKEN`. Already shipping on `aifsmjs` and `aibridgejs`; `aiecsjs` adopts in 0.2.0.
- **Publish gate**: `prepublishOnly` runs the full quality chain before allowing release; a failing gate stops the tag-triggered publish workflow.
- **Reproducible builds.** `tsup` is deterministic given the same lockfile; commit lockfiles for both `pnpm` and `npm` ecosystems as needed.

---

## License

Each package is published under the MIT License. See each package's `LICENSE` file.
