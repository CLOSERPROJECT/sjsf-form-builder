# Form composer package

Language for the standalone embeddable form composer/builder library this repo will ship. Implementation-agnostic.

## Language

**Composer**:
The embeddable UI and model that lets a host edit a form’s structure (blocks/nodes) and produce schema and uiSchema.
_Avoid_: Builder app, playground, SPA, formata-arch

**Hard fork (composer)**:
This package owns a vendored copy of upstream composer sources (`apps/builder` and required private `meta` slices) and may change them freely.
_Avoid_: Soft fork, whole-monorepo fork, “we forked svelte-jsonschema-form” (implies the entire upstream repo)

**Vendor**:
How hard-forked composer sources are stored in this repository (copied trees under our control). Not a separate upstream strategy from hard fork.
_Avoid_: Submodule-as-primary, npm package for the builder (does not exist)

**Runtime**:
Published `@sjsf/*` packages used to render or validate a form from schema/uiSchema (e.g. `@sjsf/form`, theme, validator), consumed as normal npm semver dependencies.
_Avoid_: Workspace `meta`, vendoring `@sjsf/form` sources for day-to-day use

**Upstream**:
[`x0k/svelte-jsonschema-form`](https://github.com/x0k/svelte-jsonschema-form), especially its private Form builder app (`apps/builder`) and private `meta/builder` model — not a published `@sjsf/builder`.
_Avoid_: formata-arch (archaeology only), Attesta

**Host**:
An application that depends on this package and embeds the Composer.
_Avoid_: Consumer (vague), end user (the form respondent)

**Block**:
A host- or package-defined form field/structure type editable in the Composer (the unit of registration and schema import/export).
_Avoid_: Widget-only, NodeType (upstream implementation detail unless discussing upstream maps)
