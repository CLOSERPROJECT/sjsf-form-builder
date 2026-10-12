# formata-arch: builder embed and schema import (input only)

> **Status:** archaeology for [sjsf-form-builder#4](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/4).  
> **Source pin:** `CLOSERPROJECT/formata-arch` @ `c66b464dec9b3f29ee0fdb0da4f8c7b728c98aa2` (“Confirm before save when Attesta would purge instances (#21)”).  
> **Framing:** how formata-arch embeds the builder and loads schema/uiSchema into editor state. Not product requirements; Formata/Attesta-specific contracts and field types are recorded as observed host behavior only.

Blob SHAs below are `git rev-parse c66b464:<path>` unless noted.

---

## 1. Two embed surfaces

formata-arch exposes the visual builder in two ways: an **in-process Svelte component** (`BuilderStandalone`) and a **hash-routed iframe/host page** (`#/single-form` / `#/schema-editor`). Both eventually call `BuilderContext.loadFromSchema`.

There is **no published library package**: `package.json` is `"name": "builder"`, `"private": true`, Vite app scripts only. Hosts either import source via path aliases or iframe the deployed app.

### 1.1 In-process: `BuilderStandalone`

| Piece | Path | Blob SHA |
|-------|------|----------|
| Shell (highlighter gate + context ctor) | `src/builder/builder-standalone.svelte` | `b69f4f40bb36ae42476d201b3f4433563f1df37a` |
| Layout + init effect | `src/builder/builder-standalone-content.svelte` | `fd377bc05141f4d9043607b19fe5c8e94e1c1414` |
| State / build / import | `src/builder/context.svelte.ts` | `aacf830a7817b115008222b0d45cd4b6315f2972` |

**Props surface** (`builder-standalone.svelte`):

```ts
type Props = {
  initialData?: { schema: Schema; uiSchema?: UiSchema };
  onChange?: (ctx: BuilderContext) => void;
  onInit?: (ctx: BuilderContext) => void;
};
```

Behavior (`builder-standalone-content.svelte`):

1. `setBuilderContext(ctx)`.
2. On first `$effect` tick: if `initialSchema !== undefined`, call `ctx.loadFromSchema(initialSchema, initialUiSchema)`; then `onInit?.(ctx)`.
3. Subsequent rootNode snapshot changes → `onChange?.(ctx)`.
4. Renders fixed four-column chrome: Available fields / Form structure / Field settings / Form preview (`Controls`, `Content`, `Settings`, `Form`).

**In-app consumer (workflow, not iframe):** `src/core/workflow/components/substep-formata.svelte` (`7362c7db16866f574840dac63a8be903758e52d8`) opens a dialog with:

```svelte
<BuilderStandalone
  initialData={{ schema: substep.schema, uiSchema: substep.uiSchema as UiSchema }}
  onInit={handleBuilderInit}
/>
```

Save path: host keeps `builder` from `onInit`, calls `builder.validate()`, then copies `builder.schema` / `builder.uiSchema` onto the substep. No `onChange` wiring there.

### 1.2 Host/iframe: single-form route

| Piece | Path | Blob SHA |
|-------|------|----------|
| README contract | `README.md` | `401875f170ab71a66726cd33fdb236e73a75a844` |
| App gate | `src/app.svelte` | `a72a9f0bc41264835bf2bfda98d289e5c76dbbec` |
| Route UI | `src/routes/single-form.svelte` | `2b9e595601bf214fe5e7a98cb781573f10093e60` |
| Message/HTTP helpers | `src/routes/single-form-contract.ts` | `f34794451a90c2a02f63ae8965f6d68490c7cf5b` |

`app.svelte` switches on hash: if `isSingleFormPath(hash)` → mount `SingleFormRoute`; else lazy-load the workflow router. Paths recognized: `/single-form` and `/schema-editor` (`single-form-contract.ts`).

**URL query (inside hash):** `load`, `save`, `targetOrigin` (default `window.location.origin`).

**Load paths:**

| Mechanism | Behavior |
|-----------|----------|
| `?load=<url>` | `GET` with `credentials: 'include'`, expect JSON `{ schema, uiSchema? }`. |
| Parent `postMessage` | `{ type: 'formata:schema-load', schema, uiSchema }` (origin filter is **not** applied in the listener; only message shape is checked). |
| Neither | UI stays in “waiting for schema config”. |

On mount the iframe posts `{ type: 'formata:schema-ready' }` to `targetOrigin`.

**Save path** (`single-form.svelte`):

1. Require builder present, non-empty trimmed `changeReason`, and form changed vs baseline snapshot.
2. `builder.validate()`; on failure toast and abort.
3. `builder.build()` then payload `{ schema, uiSchema, changeReason }`.
4. If `save` URL: `POST` JSON with credentials.
5. Always `postMessage({ type: 'formata:schema-saved', ...payload }, targetOrigin)`.
6. Reset baseline + clear reason.

README documents the iframe URL pattern `/formata-builder/#/single-form?load=…&save=…` and the same message types. Host owns auth/audit/persistence.

Ready data is passed into the in-process shell:

```svelte
<BuilderStandalone
  initialData={loadState.data}
  onChange={handleBuilderChange}
  onInit={handleBuilderInit}
/>
```

---

## 2. `loadFromSchema` → editor state

### 2.1 Context entry point

`BuilderContext.loadFromSchema` (`context.svelte.ts`):

```ts
loadFromSchema(schema: Schema, uiSchema?: UiSchema): void {
  const rootNode = parseSchemaToRootNode(schema, uiSchema);
  this.importState({
    rootNode,
    ignoreWarnings: this.ignoreWarnings,
    html5Validation: this.html5Validation,
    route: this.route,
    livePreview: this.livePreview
  });
}
```

`importState` assigns `rootNode` and related flags; if route is Preview it also `build()`s. Import does **not** itself run `validate()` / `build()` in the standalone init path; single-form’s `onInit` calls `ctx.validate()` after load, and live preview’s idle effect builds when valid.

Re-export: `src/lib/builder/index.ts` (`46d9724a415d7395deb8907244856ceda0c929ec`) re-exports `./schema-import.js`.

### 2.2 `parseSchemaToRootNode` (schema-import)

| Piece | Path | Blob SHA |
|-------|------|----------|
| Importer | `src/lib/builder/schema-import.ts` | `5dfcacbd0509d38cfa0361205b134740c45c9593` |

File header documents invert of `buildSchema` / `buildUiSchema`, plus **v1 limitations**:

- Conditional `schema.dependencies` with oneOf/predicates → imported as **plain object** (properties only; no dependency/predicate nodes).
- Grid → “only if detectable from uiSchema; otherwise plain object.” At this SHA there is **no** `NodeType.Grid` / grid detector in the importer body — nested objects always go through `parseObject`, so grids round-trip as ordinary groups.
- Widget round-trip is **best-effort** from `ui:components`; theme-specific options may be lost.

**Hard requirement:** root `schema.type` must be `"object"` or the parser throws.

**Dispatch order in `parseSchemaValue` (high level):**

1. Native single-file via uiSchema (`unknownField: unknownNativeFileField`) when schema has no type.
2. `enum` → `Enum` (default widget argument to infer: `radioWidget`).
3. `array` → file (data-url / ui hints), `MultiEnum` (`items.enum` + `uniqueItems`), `Tags` (`items.type === 'string'` + `uniqueItems`), else generic `Array` with recursive item.
4. `string` → file (`format: data-url` or ui `fileField`), `QrCode` (ui `stringField: formataQrField`), else `String` (date-like `format` may force `datePickerWidget`).
5. `number` / `integer` → `Number`.
6. `boolean` → `Boolean`.
7. `object` → `parseObject` (Range heuristic first: exactly `{ start, end }` with matching string or number/integer types).
8. Fallback → `String`.

Common options picked from schema: title, description, min/max items, uniqueItems, default, string/number constraints (`SCHEMA_OPTIONS_KEYS`). Help from `ui:options.help`. Property `required` is applied from the parent object’s `required` array onto child `options.required` (not from a boolean on the child schema alone, except a boolean `required` read in `applyCommonOptions`).

New node IDs are fresh `crypto.randomUUID()` values — import never preserves prior editor IDs.

---

## 3. Where per-type maps live (build vs import)

There is **no shared registry**. Build and import keep separate exhaustive tables; widget defaults are not always consistent between them.

### 3.1 Build / editor maps (emit schema + uiSchema + UI)

| Concern | Location | Blob SHA |
|---------|----------|----------|
| `NodeType` enum | `src/lib/builder/node-base.ts` | `84c2946da5ad38f17972e1c12dd3f557f414e0e0` |
| Options schemas, titles, unions | `src/lib/builder/node.ts` (+ `string-node.ts`, `number-node.ts`, `range-node.ts`, `qr-node.ts`) | `1d7cfa3c1886f803df3a8f3fbcd422889be26a0f` |
| `NODE_FACTORIES` / `createNode` | `src/lib/builder/node-factories.ts` | `956f6e359242815184b54d87847fb348f68ddce6` |
| Schema emit `Record<NodeType, …>` | `src/lib/builder/node-schema-build.ts` | `90f4acffe7729140ac2664dc5dcef846b7aa83c2` |
| UiSchema emit map | `src/lib/builder/node-ui-schema-build.ts` | `994424c648a2c639262dc83ba1fd0a0681b9b870` |
| Validate map | `src/lib/builder/node-validate.ts` | `c0fe82d80248a3f98045260cb7cc6a6d39675e28` |
| Guards | `src/lib/builder/node-guards.ts` | `841b243dddf6b2d073cda07896ae28974026fd2e` |
| Operator applicability | `src/lib/builder/detect-operators.ts` | `935f31570d32280cd5d1f163a240bdb02d4c462e` |
| Tree walk special cases | `src/lib/builder/node-traverser.ts` | `2d03e0078932afd064ad65841ceb9d6a2af33dc2` |
| `DEFAULT_COMPONENTS` / `DEFAULT_WIDGETS` (build-time ui:components) | `src/builder/model.ts` | `05771617f722581306a6cb9a6d4fd79a11301672` |
| Theme options + palette gate `THEME_CUSTOMIZABLE_NODE_TYPES` | `src/builder/theme-schemas.ts` | `ae64eeddf7836ed69c9fd923c3589bcf981dcd54` |
| Editor Svelte components `NODES` | `src/builder/nodes/index.ts` | `7ac3d1528992c16fe6f02ea0474126453011ee5c` |
| Palette icons / `createNode(t)` list | `src/builder/editor/controls.svelte` | `90453e9b37204f5c90a5125959c3fd24396f6e91` |

On build, `context.svelte.ts` derives `uiSchema` via `buildUiSchema`, injecting:

- `DEFAULT_COMPONENTS[resolver][node.type](node)`
- override `components[DEFAULT_WIDGETS[type]] = widget` when widget ≠ default

Palette membership is **theme-gated**: `availableCustomizableNodeTypes = THEME_CUSTOMIZABLE_NODE_TYPES[theme]` (Shadcn4 list includes Range; basic list in the same file does not).

### 3.2 Import-only maps (invert ui:components → node)

Inside `schema-import.ts` only:

| Map | Role |
|-----|------|
| `BUILDER_DEFAULT_WIDGETS` | Expected default widget names for override detection (`String`→`textWidget`, `Enum`→`selectWidget`, …). Partial; no File/Tags/Range/QrCode entries. |
| `WIDGET_OVERRIDES` | Allowed non-default widgets per type when reading `ui:components[defaultWidget]`. |
| `inferWidgetFromUiSchema` | Heuristics: `formataQrField`, `aggregatedField`, `multiEnumField`, `arrayTagsField`, file field names, then override lookup. |
| `getFileFieldFromUiSchema` / `isQrCodeFromUiSchema` | Structural field-kind detection from `ui:components`. |

These maps **mirror** build emissions in `model.ts` `DEFAULT_COMPONENTS` / `DEFAULT_WIDGETS` by convention, not by shared constant.

### 3.3 Observed asymmetries (facts at pin)

- **Enum default widget:** `ENUM_OPTIONS_SCHEMA` and `DEFAULT_WIDGETS` / `BUILDER_DEFAULT_WIDGETS` use `selectWidget`; `createNode(Enum)` and import’s `inferWidgetFromUiSchema(…, 'radioWidget')` call site seed **`radioWidget`**. Round-trip can change the stored widget unless uiSchema encodes an override.
- **Tags in `DEFAULT_WIDGETS`:** `[NodeType.Tags]: FILE_NODE_OPTIONS_SCHEMA.properties.widget.default` (file default), while `TAGS_NODE_OPTIONS_SCHEMA` defaults to `tagsWidget`. Affects build-time “is this an override?” comparison, not the Tags factory default.
- **Grid:** schema builder emits grid layouts; importer has no grid reconstruction path at this SHA despite the file comment.
- **Dependencies / predicates:** built by editor + `node-schema-build`; stripped to plain properties on import.

---

## 4. Multi-file block registration pain (observed)

Adding a customizable `NodeType` is an exhaustive multi-file edit. There is no `defineBlock` / `registerBlocks` API in this tree.

**Minimum touch set for a leaf like QrCode** (verified present at pin):

1. `node-base.ts` — enum member  
2. Dedicated options module (e.g. `qr-node.ts`) + `index.ts` re-export  
3. `node.ts` — interface, options schema, `CUSTOMIZABLE_TYPE_TITLES`, `NODE_OPTIONS_SCHEMAS`, `NODE_OPTIONS_UI_SCHEMAS`  
4. `node-factories.ts` — `NODE_FACTORIES` entry  
5. `node-guards.ts`  
6. `node-validate.ts` map  
7. `node-schema-build.ts` map  
8. `node-ui-schema-build.ts` map  
9. `detect-operators.ts` map  
10. `schema-import.ts` — recognition heuristics (Qr uses `formataQrField`)  
11. `builder/nodes/index.ts` — `NODES`  
12. `editor/controls.svelte` — `NODE_ICONS`  
13. `theme-schemas.ts` — theme option factories / customizable list  
14. `model.ts` — `DEFAULT_COMPONENTS` + `DEFAULT_WIDGETS`  

Composite types (Range) additionally need traverser / props / dedicated Svelte + theme range enums (`node-traverser.ts`, `node-props.ts`, `nodes/range.svelte`, `THEME_RANGE_VALUE_TYPES`).

TypeScript’s `Record<NodeType, …>` exhaustiveness catches missed cases at compile time; it does not provide a single registration seam.

First-party write-up of the same inventory (design note, not runtime): [formata-arch#22](https://github.com/CLOSERPROJECT/formata-arch/issues/22) §1.4.

---

## 5. End-to-end data flow (summary)

```text
Host schema + uiSchema
        │
        ├─ iframe: GET load / formata:schema-load  → SingleFormRoute
        └─ Svelte: initialData / substep props
                │
                ▼
        BuilderStandalone → loadFromSchema
                │
                ▼
        parseSchemaToRootNode  →  rootNode (editor state)
                │
        edit / DnD / settings
                │
                ▼
        validate() + build() + derived buildUiSchema
                │
                ▼
        schema + uiSchema  →  host save / postMessage / substep assign
```

---

## 6. Sources

| Ref | What it owns |
|-----|----------------|
| `c66b464dec9b3f29ee0fdb0da4f8c7b728c98aa2` | Tree pin for all path/blob citations above |
| Blob SHAs in §§1–4 | Exact file contents at pin |
| [formata-arch#22](https://github.com/CLOSERPROJECT/formata-arch/issues/22) | First-party narrative of multi-touch registration (corroborates §4) |
| [sjsf-form-builder#4](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/4) | Research question this note answers |

No secondary blogs or DeepWiki pages were used for claims in this document.
