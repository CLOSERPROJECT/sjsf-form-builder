# Research: Upstream relationship options for the form-builder package

**Ticket:** [CLOSERPROJECT/sjsf-form-builder#2](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/2)  
**Map:** [CLOSERPROJECT/sjsf-form-builder#1](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/1)  
**Researched:** 2026-10-12  
**Status:** Options and criteria only — **no product decision locked here** (grilling ticket owns the choice).

## Question

What is the best **upstream relationship** between this package and [`x0k/svelte-jsonschema-form`](https://github.com/x0k/svelte-jsonschema-form) (especially `apps/builder` and private `meta/builder`)?

Compare:

1. Hard-fork / vendor builder sources + npm `@sjsf/*` runtime deps  
2. Thin wrapper / submodule / alias over upstream `apps/builder`  
3. Wait for an official published builder package  
4. Greenfield composer UI on `@sjsf/form` only  

For each: feasibility today, maintenance cost, impact on host block registration + schema import/export, and **recommendation criteria** (not a pick).

## Method and pins

| Item | Value |
|------|--------|
| Upstream repo | https://github.com/x0k/svelte-jsonschema-form |
| Pin SHA (issue authoring tip) | [`703c86bc2f75bf808181900a1e37929111cafd7e`](https://github.com/x0k/svelte-jsonschema-form/commit/703c86bc2f75bf808181900a1e37929111cafd7e) |
| Tip vs pin (2026-10-12) | **Identical** (`compare` status `identical`, ahead/behind 0) |
| npm sample | `@sjsf/form@3.9.0` (latest at research time) |
| Archaeology only | [formata-arch#22](https://github.com/CLOSERPROJECT/formata-arch/issues/22) — prior hard-fork intent; **not** authority for this repo |

Primary sources used: upstream GitHub tree/manifests/source at the pin SHA, npm registry responses, Changesets config, monorepo contributing skill, MIT `LICENSE`, formata-arch#22 and formata-arch tree paths only as archaeology.

---

## Facts about upstream today

### Layout: libraries vs apps

Upstream is a pnpm monorepo. Official contributing notes classify:

- `packages/*` — published libraries (`@sjsf/form`, themes, validators, …)  
- `apps/*` — sites: `docs2`, `playground2`, **`builder` (Form builder app)**  
- `lab/*` — experimental packages  

Source: [`skills/sjsf-monorepo-dev/references/contributing-workflows.md`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/skills/sjsf-monorepo-dev/references/contributing-workflows.md) at pin.

At pin: **299** files under `apps/builder/`; **18** files under `packages/meta/src/builder/`.

### `apps/builder` is a private Vite SPA, not a package

[`apps/builder/package.json`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/package.json):

- `"name": "builder"`, `"private": true`, version `0.8.4`  
- Scripts: `vite` / `vite build` / `svelte-check` — **no** `@sveltejs/package` / library `exports`  
- Depends on `"meta": "workspace:*"` plus many `workspace:*` `@sjsf/*` and `@sjsf-lab/*` packages  
- Multi-theme / multi-validator playground surface is baked into dependencies  

[`apps/builder/vite.config.ts`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/vite.config.ts):

- `base: "/svelte-jsonschema-form/builder3/"` (deployed site path)  
- aliases `$lib` → `./src/lib`  
- imports `importMapPlugin` from `meta/vite-importmap`  

Entry is a full app shell: [`main.ts`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/src/main.ts) mounts [`app.svelte`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/src/app.svelte) (IndexedDB projects, header chrome, toaster, theme manager) — not an embeddable library export.

Docs/README link the builder as a **hosted site** (`…/builder3`), not an installable package ([root README](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/README.md)).

### Private `meta` / `meta/builder` is required by the builder

[`packages/meta/package.json`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/packages/meta/package.json):

- `"name": "meta"`, **`"private": true`**  
- Subpath export `"./builder"` → `dist/builder`  
- Also exports `playground`, `sandbox`, `composer`, `codegen`, … used by apps and tooling  

npm: **`@sjsf/meta` → 404**; there is no published `meta` package under the `@sjsf` scope.

At pin, `apps/builder` import sites (ripgrep on clone of pin):

| Import | Count (approx.) |
|--------|-----------------|
| `meta/builder` | 36 |
| `meta/playground` | 8 |
| `meta/sandbox` | 2 |
| `meta/vite-importmap` | 1 (vite config) |

[`packages/meta/src/builder/index.ts`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/packages/meta/src/builder/index.ts) re-exports the shared node model: `NodeType`, options schemas, widgets, operators, sandbox helpers, etc.

**Implication:** anything that consumes upstream builder sources must also obtain **`meta/builder` (and currently playground/sandbox slices)** somehow — npm alone is insufficient.

### Published npm surface (no builder)

Verified on registry (2026-10-12):

| Package | Status |
|---------|--------|
| `@sjsf/form`, themes (`basic`, `shadcn4`, `daisyui5`, `flowbite3`, …), validators (`ajv8`, `zod4`, `valibot`, …), icons, `@sjsf/sveltekit` / `sveltekit3` | **Published** (form/themes/validators cluster at **3.9.0**) |
| `@sjsf/builder` | **404** |
| `@sjsf/form-builder` | **404** |
| `@sjsf/meta` | **404** |

Changesets [`.changeset/config.json`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/.changeset/config.json) `fixed` array lists published `@sjsf/*` libraries — **neither `builder` nor `meta`**. `privatePackages` only versions private pkgs for internal bumps (`tag: false`).

No open upstream issue was found that proposes publishing `@sjsf/builder` as a library (search at research time). Closed V3 tracking ([#165](https://github.com/x0k/svelte-jsonschema-form/issues/165)) is about form runtime breaking changes, not a builder package.

License: MIT ([`LICENSE`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/LICENSE)); `@sjsf/form` additionally notes Apache-2.0 for RJSF-derived parts (root README).

### Block registration model (multi-touch maps)

Upstream does **not** expose a plugin/`defineBlock` API. Types and UI are wired through exhaustive maps keyed by `NodeType` / `CustomizableNodeType`, split across **private meta** and **app** code. Examples at pin:

- Enum + titles / options: `packages/meta/src/builder/node-types.ts`, `node-types-full.ts`, `customizable-node-types.ts`  
- Factories: [`apps/builder/src/lib/builder/node-factories.ts`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/src/lib/builder/node-factories.ts) (`NODE_FACTORIES: { [T in CustomizableNodeType]: … }`)  
- Guards: [`node-guards.ts`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/src/lib/builder/node-guards.ts)  
- Schema / UI schema builders: [`node-schema-build.ts`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/src/lib/builder/node-schema-build.ts), [`node-ui-schema-build.ts`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/src/lib/builder/node-ui-schema-build.ts) (`NODE_UI_SCHEMA_BUILDERS: { [T in NodeType]: … }`)  
- UI components: [`apps/builder/src/builder/nodes/index.ts`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/src/builder/nodes/index.ts) (`NODES` map)  
- Options schemas table: [`node.ts`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/src/lib/builder/node.ts) (`NODE_OPTIONS_SCHEMAS`)  
- Theme gating / palette: `theme-schemas.ts`, editor `controls.svelte`, `model.ts` defaults  

Adding a host-defined block type against this shape means touching **many** of those tables (TypeScript exhaustiveness helps safety, hurts drop-in plugins). Charting preference for a single-descriptor registry is a **divergence** from upstream, not a thin config knob.

### Schema import / export

**Export (nodes → JSON Schema / uiSchema):** first-class in upstream. `buildSchema` / `buildUiSchema` are used from `BuilderContext` to drive preview and playground samples ([`context.svelte.ts`](https://github.com/x0k/svelte-jsonschema-form/blob/703c86bc2f75bf808181900a1e37929111cafd7e/apps/builder/src/builder/context.svelte.ts)).

**Persistence:** projects store **`BuilderState`** (primarily `rootNode` + theme/validator/route flags), not a round-trip through arbitrary JSON Schema alone (`exportState` / `importState` on `BuilderContext`; IndexedDB via `projects/`).

**Import (arbitrary JSON Schema → node tree):** no dedicated `schema-import` module under `apps/builder` or `packages/meta/src/builder` at pin (search for import/deserialize helpers did not surface a schema→nodes pipeline).  

**Archaeology:** formata-arch at cited tip contains `src/lib/builder/schema-import.ts` and embed shells `builder-standalone*.svelte` ([formata-arch tree](https://github.com/CLOSERPROJECT/formata-arch); described in [formata-arch#22](https://github.com/CLOSERPROJECT/formata-arch/issues/22)). Treat as prior art only — contaminated with Attesta/Formata goals and older relative to upstream tip.

---

## Option comparison

### 1. Hard-fork / vendor builder + npm `@sjsf/*` runtime

**What it means:** Copy (and prune/transform) `apps/builder` sources **and** needed private `meta` slices (at least `meta/builder`; likely playground/sandbox helpers used by builder) into this repo. Depend on published `@sjsf/form` (+ one theme + one validator) via semver. Do **not** submodule the whole monorepo for day-to-day imports.

| Dimension | Assessment |
|-----------|------------|
| **Feasibility today** | **High.** MIT allows copy; pin is current; runtime packages exist on npm; builder+meta sources are readable in-tree. Must invent package boundary (embed API, strip SPA chrome / IDB / multi-theme lab deps). |
| **Maintenance cost** | **Medium–high.** Own the forked tree; run pin-and-diff intake. Cost spikes if host-block registry rewrite diverges deeply (expected per map charting). Sync is selective (bring/adapt/skip), not clean merges of whole trees. |
| **Host block registration** | **Requires structural change** on the fork (registry / `defineBlock`). Upstream maps are not plugin-friendly; forking is what makes the rewrite *possible* without fighting a live submodule. |
| **Schema import/export** | **Export:** can keep/adapt `buildSchema` / `buildUiSchema`. **Import:** not complete upstream — implement or adapt from archaeology; per-block serialize/deserialize aligns with a registry rewrite. |

**Recommendation criteria (when this option scores well):** need composer UX close to upstream soon; accept owning diverged sources; willing to vendor private `meta/builder`; registry rewrite is a goal; shipping must not wait on upstream packaging.

---

### 2. Thin wrapper / submodule / alias over `apps/builder`

**What it means:** Pin or submodule `x0k/svelte-jsonschema-form@SHA`, Vite/TS-path alias into `apps/builder/src/...` and `packages/meta`, re-export a thin embed API without copying sources into this package’s forever-tree.

| Dimension | Assessment |
|-----------|------------|
| **Feasibility today** | **Partial / fragile.** Technically possible in a monorepo-aware bundler setup, but you are packaging a **private app** that expects workspace `meta`, `$lib`, GH-pages `base`, and playground meta packages. Consumers cannot `npm install` upstream builder. |
| **Maintenance cost** | **High for “stay thin.”** Every upstream path/`$lib`/meta export churn breaks the wrapper. After a registry rewrite, the “thin” layer becomes a de facto fork while still fighting submodule layout — **worst of both**. |
| **Host block registration** | **Blocked or forced fork.** Exhaustive maps live upstream; host plugins need either upstream cooperation or local source edits (which collapses to option 1). |
| **Schema import/export** | Same upstream capabilities/gaps as option 1, but harder to extend import/export seams without patching submodule files. |

**Recommendation criteria (when this option scores well):** short spike / prototype only; zero intent to rewrite registry; acceptable to ship only as an internal app that builds against the full upstream workspace; willing to drop the approach once structural edits start.

---

### 3. Wait for official `@sjsf/builder` (or similar)

**What it means:** Do not vendor or wrap; track upstream until a published, documented embeddable builder package exists; then depend on it.

| Dimension | Assessment |
|-----------|------------|
| **Feasibility today** | **None for shipping.** `@sjsf/builder` / `@sjsf/form-builder` / `@sjsf/meta` are unpublished; Changesets do not release the builder app; docs treat Form Builder as a website. No primary-source commitment to publish was found at research time. |
| **Maintenance cost** | **Low while waiting; schedule risk high.** Product timeline is gated on a third party. If/when published, maintenance could become lowest of all options — **if** the published API matches host-registration and schema I/O needs. |
| **Host block registration** | **Unknown / likely insufficient initially.** Upstream’s current model is not plugin-oriented; a future package would need a new public API for host blocks. |
| **Schema import/export** | **Unknown.** Today only node→schema build is solid; a future package may or may not expose per-block I/O. |

**Recommendation criteria (when this option scores well):** no near-term ship date; willingness to re-evaluate if upstream announces a library; product can live with `@sjsf/form` alone or a temporary greenfield stub until then. Charting already **deferred** this as non-blocking research.

---

### 4. Greenfield UI on `@sjsf/form` only

**What it means:** Implement a new composer UI and node/block model from scratch. Use `@sjsf/form` (+ theme/validator) only for preview/runtime rendering of JSON Schema / uiSchema. No copy of `apps/builder` / `meta/builder`.

| Dimension | Assessment |
|-----------|------------|
| **Feasibility today** | **High for runtime; full rewrite for composer.** npm `@sjsf/form@3.9.0` is the supported product. All DnD, grid, predicates/operators, builder-specific widgets must be re-specified. |
| **Maintenance cost** | **High upfront, then independent.** No upstream builder sync. Ongoing cost is owning UX/semantics; runtime upgrades follow normal `@sjsf/*` semver. |
| **Host block registration** | **Best blank canvas.** Can design single-descriptor registration without fighting exhaustive upstream maps. |
| **Schema import/export** | **Design freedom / no free builder pipeline.** Must implement node↔schema (or block↔schema) themselves; cannot reuse `buildSchema` without taking builder sources (option 1). |

**Recommendation criteria (when this option scores well):** upstream builder semantics are a poor fit; registry/plugin DX and clean package API matter more than feature parity with builder3; team accepts rewriting DnD/conditional/grid behavior; desire to avoid private `meta` entirely.

---

## Cross-cutting notes (all options)

1. **Runtime vs composer split is already real upstream:** `@sjsf/*` libraries are the supported embed story; the visual builder is an app. Any “relationship” decision is really about the **composer** layer.  
2. **Private `meta/builder` is the hidden dependency** of options 1–2. Option 4 avoids it; option 3 waits for upstream to productize it.  
3. **Map charting preferences that interact with this research** (not locked by this ticket): host-registerable blocks; per-block schema I/O; single theme/validator; embeddable composer not SPA chrome; default live preview via `@sjsf/form` or host-replaceable renderer ([#1](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/1)). Options that cannot meet those preferences without deep edits effectively **require** forking or greenfield.  
4. **formata-arch#22** previously preferred hard-fork + npm runtime and rejected submodule-as-primary — useful archaeology, **not** a decision for this map.

---

## Recommendation criteria (decision aid — not a pick)

Use a grilling ticket to lock one option. Suggested evaluation axes:

| Criterion | Favors |
|-----------|--------|
| Ship an embeddable composer **this quarter** with upstream-like UX | 1 (fork), maybe short-lived 2 for spikes |
| Minimize long-term sync against private app paths | 4, or 3 if waiting is acceptable |
| Host `defineBlock` / single-descriptor registry is non-negotiable | 1 (with rewrite) or 4; **not** thin 2; **not** wait-and-hope 3 |
| Per-block schema import/export is first-class | 1 or 4 (both need work beyond upstream); 2 makes extension awkward |
| Avoid vendoring / license attribution surface area | 3 or 4 |
| Keep `@sjsf/*` as normal semver deps for **preview/runtime** | 1, 3 (if published), 4 — all fine; 2 often pulls workspace wiring |
| Upstream later publishes `@sjsf/builder` with a real public API | Re-open relationship (could migrate from 1→dep or 4→dep) |

**This document does not select among (1)–(4).**

---

## Sources

1. Upstream pin commit: https://github.com/x0k/svelte-jsonschema-form/commit/703c86bc2f75bf808181900a1e37929111cafd7e  
2. `apps/builder/package.json`, `vite.config.ts`, `src/app.svelte`, `src/main.ts`, `src/lib/builder/*`, `src/builder/nodes/index.ts`, `src/builder/context.svelte.ts` at pin  
3. `packages/meta/package.json`, `packages/meta/src/builder/*` at pin  
4. `pnpm-workspace.yaml`, root `package.json`, `.changeset/config.json`, `LICENSE`, root `README.md` at pin  
5. Contributing layout notes: `skills/sjsf-monorepo-dev/references/contributing-workflows.md` at pin  
6. npm registry: `@sjsf/form@3.9.0`; 404 for `@sjsf/builder`, `@sjsf/form-builder`, `@sjsf/meta` (queried 2026-10-12)  
7. Upstream issues: [#165 V3](https://github.com/x0k/svelte-jsonschema-form/issues/165) (no builder publish plan)  
8. Archaeology: [formata-arch#22](https://github.com/CLOSERPROJECT/formata-arch/issues/22); formata-arch paths `src/lib/builder/schema-import.ts`, `src/builder/builder-standalone*.svelte`  
9. Map context: [sjsf-form-builder#1](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/1), research ticket [#2](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/2)
