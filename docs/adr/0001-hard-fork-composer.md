# Hard-fork the composer; npm for runtime

We need an embeddable form **Composer** with host-registerable blocks and per-block schema I/O. Upstream publishes `@sjsf/*` runtime libraries but not a builder package: `apps/builder` is a private Vite SPA that depends on private `meta` (including `meta/builder`). Thin wrapping that app is fragile under a registry rewrite; waiting for `@sjsf/builder` is not shippable; greenfield would redo DnD/node semantics we can transform instead.

**Decision:** Hard-fork the composer — vendor `apps/builder` plus required `meta` slices at an upstream pin, transform those sources in place into a library, and keep `@sjsf/form` (theme, validator) as npm semver deps only. Stay upstream-aware with pin + incremental bring/adapt/skip; re-open if an official builder package appears.

**Rejected:** submodule/alias wrapper as the primary strategy; blocking on an official builder; greenfield composer; forking or vendoring the whole upstream monorepo for runtime.

See wayfinder ticket [Choose the upstream relationship for this package](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/5) and research [upstream-relationship.md](https://github.com/CLOSERPROJECT/sjsf-form-builder/blob/research/upstream-relationship/docs/research/upstream-relationship.md).
