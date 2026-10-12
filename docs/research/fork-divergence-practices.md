# Research: re-appliable hard-fork / vendor-overlay divergence tracking

**Ticket:** [#3](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/3)  
**Map:** [#1](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/1)  
**Date:** 2026-10-12  
**Scope:** Survey established practices that make a hard fork or vendor overlay **safe to re-apply** when upstream moves — especially tracking unmodified vs lightly patched vs heavily replaced files. **No final product choice** (grilling ticket later).

## Question framing

For a Svelte package that may deeply rewrite an upstream block registry, “safe to re-apply” means:

1. Upstream can move (new commits/tags) without silently discarding fork intent.
2. Maintainers can tell, **before** syncing, which paths are pristine, thin-patched, or effectively replaced.
3. Sync failures are local and recoverable (named conflict, regenerable artifact), not “whole tree mush.”
4. Divergence cost is visible enough that a later grilling ticket can pick among options.

---

## Pattern catalog

Each pattern below: **what it is**, **sync flow**, **how divergence classes surface**, **failure modes**, **fit for deep Svelte registry rewrite**. Citations are primary docs/repos.

### 1. Quilt / Debian `3.0 (quilt)` patch series

**What.** Ordered stack of unified diffs plus a `series` file; apply/unapply as a stack (`push`/`pop`). Debian packages store this under `debian/patches/` and declare `3.0 (quilt)` so `dpkg-source` applies patches with **no fuzz** at extract time.[^quilt-man][^dpkg-source][^debsrc30]

**Sync flow.**

1. Obtain new upstream tarball / tree.
2. `quilt push -a` (or let `dpkg-source` apply) on the new base.
3. On reject: edit, `quilt refresh`, or reorder/drop patches in `series`.
4. Ship refreshed patch files + `series` with the packaging tree.

**Divergence classes.**

| Class | How it appears |
| --- | --- |
| Unmodified | Path never appears in any patch hunk |
| Lightly patched | Small numbered patches; DEP-3 headers can document intent |
| Heavily replaced | Large / many patches touching one path; still a *delta*, not a dual file |

**Failure modes.** Fuzz rejected (`dpkg-source` requires clean apply); context drift after refactors; mega-patches that are effectively unmaintainable; accidental merge of all local edits into one catch-all `debian-changes-*` patch if changes were not quilted first.[^debsrc30]

**Fit for Svelte registry rewrite.** Strong for **many small, ordered** tweaks to upstream files. Weak when a registry is rewritten wholesale — the “patch” becomes a full-file replace in disguise, and every upstream rename/refactor re-breaks the series. Good as a *component* of a vendor flow, not as the sole home for a deep rewrite.

---

### 2. Vendor + patches (OpenTitan `vendor.py`, Bender)

**What.** Copy a pinned upstream revision into-tree; keep local edits as numbered `.patch` files applied after copy. OpenTitan’s `util/vendor.py` is the reference hardware-monorepo design; Bender’s `vendor` command is explicitly “heavily inspired by” that script.[^opentitan-vendor][^bender-vendor]

**OpenTitan specifics.**

- `.vendor.hjson` + `.lock.hjson` pin URL/rev; lock makes re-import reproducible without `--update`.[^opentitan-vendor]
- `patch_dir` with lexicographically ordered `0001-….patch`; applied with `git apply -p1`.[^opentitan-vendor]
- Optional `patch_repo` + `--refresh-patches`: export `rev_base..rev_patched` commits from a fork into the patch directory (Git as the editor; patches as the durable product).[^opentitan-vendor]
- Policy: prefer upstream-first; patches when upstream is slow, refuses, or change is project-specific.[^opentitan-vendor]

**Bender specifics.**

- `bender vendor init` → clone/filter/copy → apply `patch_dir`.
- Edit in `target_dir`, `bender vendor diff`, `bender vendor patch` → numbered patch; commit **both** modified tree and patch file.
- Teammates re-run `init` to rebuild from pin + patches.[^bender-vendor]

**Sync flow.** Bump upstream rev in config → re-run vendor (with update) → patches re-apply → on failure refresh patches (OpenTitan: rebase in `patch_repo` then `--refresh-patches`; Bender: fix tree and regenerate patch).

**Divergence classes.**

| Class | How it appears |
| --- | --- |
| Unmodified | Present in vendored tree, absent from all patches |
| Lightly patched | Small ordered patches; `vendor diff` / patch review |
| Heavily replaced | Huge patches or many overlapping patches on one path — still delta-shaped |

**Failure modes.** Patch apply fails after upstream move; forgetting to commit regenerated patches; editing vendored files without capturing a patch (drift between tree and recipe); include/exclude globs silently dropping needed files.[^opentitan-vendor][^bender-vendor]

**Fit.** Excellent when the fork is mostly **upstream copy + thin deltas**. A deep registry rewrite will inflate patch surface unless the rewrite lives *outside* the vendored tree (see erigone overlay). OpenTitan’s `patch_repo` pattern is useful if maintainers prefer editing on a real Git fork then exporting patches.

---

### 3. Vendor + patches (JS): `inrepo`

**What.** npm-oriented CLI: pin upstream git commit in `inrepo.lock.json`, keep changes under `inrepo_patches/`, rebuild `inrepo_modules/` with `inrepo sync`. Formula: `upstream commit + committed patches = generated local package`.[^inrepo]

**Two patch formats.**

1. **Series:** `inrepo_patches/<pkg>/series/0001-*.patch` — `git format-patch --binary`, applied with `git am --3way` in filename order (no separate series file).[^inrepo]
2. **Legacy snapshots:** whole-file overlays + `.inrepo-deletions` (still supported; migrate to series for rebaseable updates).[^inrepo]

**Sync / update flow.**

- Day-to-day: edit `inrepo_modules/` → `inrepo patch -m "…"` appends next series patch → commit patches + lock.
- Upstream bump: `inrepo update` rebases the series onto the new commit in a scratch repo; drops patches upstream absorbed; on conflict keeps worktree under `.inrepo/updates/…` with `--continue` / `--abort`.[^inrepo]
- CI: `inrepo verify` catches uncaptured edits / drift.[^inrepo]

**Divergence classes.**

| Class | How it appears |
| --- | --- |
| Unmodified | No patch hunks; `inrepo diff --stat` quiet for that path |
| Lightly patched | Short series entries with subjects as provenance |
| Heavily replaced | Long series or (legacy) whole-file snapshots — snapshots are explicitly non-rebaseable until migrated[^inrepo] |

**Failure modes.** Uncaptured edits in `inrepo_modules/` (sync refuses); simultaneous overlay + module edits (conflict); series rebase conflict on update; empty dirs cannot ride in a git series.[^inrepo]

**Fit.** Strong candidate shape for a **Svelte/npm package**: in-repo visibility, lock pin, rebaseable series, verify gate. Deep registry rewrite still hurts as pure patches; better if rewritten registry code is **owned overlay** (new files) and only thin wiring patches touch upstream. `repositoryDirectory` helps when upstream is a monorepo package.[^inrepo]

---

### 4. Overlay + thin patches (erigone-style)

**What.** ethpandaops/erigone does **not** keep a full Erigon fork. Repo stores:

- `overlay/` — **new** files copied into a fresh upstream clone (build-tagged so they do not collide).
- `patches/<org>/<repo>/<ref>/` — minimal `base.patch` (+ optional `01-….patch`) per upstream ref.
- Scripts: clone → apply patches → copy overlay → build; CI daily-checks apply + auto-commits 3-way patch refreshes when possible.[^erigone]

**Design rule (stated tip):** most features are overlay + maybe a dependency script; touching upstream files should be rare — less patch surface, fewer sync conflicts.[^erigone]

**Sync flow.**

1. Build against new upstream ref (`erigone-build.sh -b <ref>`).
2. If patch hunks fail: fix in clone → `save-patch.sh` regenerate → commit patches.
3. Multi-ref support: port patch changes per ref subdirectory; overlay variants (`*_main` / `*_v3`) selected by a `variant` file.[^erigone]

**Divergence classes.**

| Class | How it appears |
| --- | --- |
| Unmodified | Never in patches; not overwritten by overlay |
| Lightly patched | Thin wiring in `base.patch` / extension patches |
| Heavily replaced | Prefer **duplicate logic in overlay** over patching large upstream functions (explicit tradeoff in their gas-sim work: duplicate rather than widen patch surface)[^erigone-commit] |

**Failure modes.** Patch conflict on upstream refactor; forgetting to port a change to every supported ref; overlay API drift across variants; CI hard-fail when 3-way merge cannot recover.[^erigone]

**Fit.** **Best structural match** if the form-builder’s value is a new/rewritten registry and preview seams: keep rewrite as overlay (or sibling package), patch only registration/wiring hooks in upstream. Mirrors the charting preference to treat product surface as composer + registry + seams rather than upstream SPA chrome ([#1](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/1)).

---

### 5. Divergence ledgers (otto-cli `UPSTREAM-SYNC`)

**What.** Permanent hard fork of vendored trees, plus a **living Markdown ledger** that records:

1. Fork baseline (upstream project, version, import SHA, date, method).
2. Per-package divergence / risk table (Heavy / Moderate / Minimal / Wrapper-only).
3. File-level patch log (path → commits + rationale).
4. Sync workflow: cherry-pick / manual port / skip, with rules to update the ledger in the same commit.[^otto-sync]

Companion pipeline docs automate candidate discovery and swarm stages; the ledger remains the human contract for “what we changed and why.”[^otto-pipeline]

**Sync flow.**

1. `git fetch upstream`; log since baseline on care paths.
2. For each commit: read patch, compare to local divergence, consult ledger risk.
3. Apply via clean cherry-pick, manual port (cite upstream SHA), or skip (note in ledger).
4. Update divergence table + patch log; run build/tests.[^otto-sync]

**Divergence classes.**

| Class | How it appears |
| --- | --- |
| Unmodified / minimal | Package row “Minimal”; empty or branding-only patch log |
| Light / moderate | Explicit Moderate row + file entries |
| Heavy | **Heavy** + high sync risk; file-level log required before touching |

Classification is **maintainer-authored metadata**, not mechanically derived from the tree (unlike patch hunk presence).

**Failure modes.** Ledger drift (forgot to update → next sync flies blind); silent cherry-picks without SHA citation; treating Heavy packages as clean cherry-picks; branding auto-scripts confused with behavioral patches.[^otto-sync]

**Fit.** Essential **companion** when the relationship is a true hard fork (full tree copy). Alone it does not make re-apply mechanical — it makes risk **legible**. For a deep registry rewrite, a Heavy row on registry paths is exactly the signal a grilling ticket needs. Pair with patches/overlay so the ledger is not the only recovery tool.

---

### 6. Dual-file / `*.custom.*` (and `.local` overlay) conventions

**What.** Keep upstream- or generator-owned baseline files refreshable; put fork/user intent in a **sibling file** that sync tools refuse to overwrite.

**Primary examples.**

- **ampless:** `backend.custom.ts` / `resource.custom.ts` — baseline calls into customize hooks; `create-ampless upgrade` protects `*.custom.ts` from overwrite.[^ampless-custom]
- **clowder-ai:** `foo.local.md` (append) vs `foo.local-override.md` (replace); overlays gitignored so fork users customize without merge conflicts on tracked rules.[^clowder-local]
- **MakeCode / PXT:** dedicated `custom.ts` for user blocks, separate from `main.ts`, so custom definitions are not treated as grey/orphan blocks.[^pxt-custom]

**Sync flow.** Refresh/regenerate baseline files from upstream or scaffold; leave `*.custom.*` / `.local*` untouched; resolve by import/hook/merge rules at build or runtime.

**Divergence classes.**

| Class | How it appears |
| --- | --- |
| Unmodified | Baseline file only; no sibling |
| Lightly patched | N/A as patch — instead “thin hook” in baseline calling custom |
| Heavily replaced | `*.local-override` / full custom module that replaces behavior |

**Failure modes.** Baseline must expose stable hooks or customization cannot attach without editing baseline (which reintroduces merge pain); naming convention without toolchain enforcement; gitignored overlays invisible to teammates/CI if the customizations were meant to be shared.[^ampless-custom][^clowder-local]

**Fit.** Strong when the registry API can be **extended** via host registration / customize hooks (aligned with charting preference for host-registerable blocks). Weak if the rewrite must mutate deep internals of upstream modules that expose no extension point — then dual-file cannot replace patches/overlay wiring.

---

### 7. Git overlay branches (and vendor-branch replay)

**What.** Keep a clean upstream-tracking lane and a **single long-lived overlay branch** of local-only commits; rebase overlay onto upstream when it moves. Documented operator pattern: Routa’s local overlay + upstream sync guide.[^routa-overlay]

**Routa branch model.**

1. `upstream/main` — fetch only, never local edits.
2. `local/routa-overlay-*` — only long-lived local lane; rebased onto upstream.
3. Temporary PR branches — upstreamable work; deleted after merge/rejection.
4. Prefer two worktrees: clean upstream vs overlay.[^routa-overlay]

**Sync flow.**

```bash
git fetch upstream
git switch main && git reset --hard upstream/main
git switch local/routa-overlay-…
git rebase upstream/main
# resolve overlay conflicts; drop commits upstream absorbed
```

**Related tooling.** `git-vendor-replay` records pristine vendor imports (`Vendor-dir:` trailer) and prepares replay of downstream commits onto a new import — explicit review of every downstream patch each bump; suggests subtree merge when you want connected upstream history instead of independent imports.[^git-vendor-replay]

**Divergence classes.**

| Class | How it appears |
| --- | --- |
| Unmodified | No overlay commits touch the path (`git log overlay ^upstream -- path`) |
| Lightly patched | Few small overlay commits |
| Heavily replaced | Large overlay commits / many files; still recoverable via rebase conflict locality |

**Failure modes.** Mixing local-only and upstreamable work on the overlay branch; force-push pain on shared overlay; rebase conflict explosives if overlay is not kept thin; machine-local state accidentally committed.[^routa-overlay]

**Fit.** Natural if the product **is** a GitHub fork of `x0k/svelte-jsonschema-form` (or a subtree). Deep registry rewrite as a few overlay commits is reviewable, but history rewrite + force-push discipline is operational cost. Pair with a ledger for Heavy paths so agents/humans know where rebase pain will concentrate.

---

### 8. Package-manager patches (Yarn `patch:` / pnpm `patchedDependencies`) — adjacent, not a full fork model

**What.** Official package-manager support for storing a patch applied at install time to a registry dependency — Yarn `patch:` protocol + `yarn patch` / `patch-commit`;[^yarn-patch] pnpm `pnpm patch` / `patch-commit` registering `patchedDependencies`, with explicit failure / unused-patch controls.[^pnpm-patch]

**Sync flow.** Bump dependency version → often regenerate patch against new extract → install fails if patch no longer applies (unless ignore/allow flags).

**Divergence classes.** Entire package is either unpatched or “has this patch file”; no built-in per-file Heavy/Light ledger (third-party sidecars like patchlift exist for Yarn patch metadata, but are not the package manager itself).

**Fit.** Appropriate for **consumer apps** fixing a published builder. **Not** sufficient as the divergence model for *being* the hard-forked package — there is no first-class overlay, multi-ref matrix, or fork baseline. Useful as a comparison point: same failure mode (context drift) at much smaller scope.

---

## Cross-cutting: unmodified vs light vs heavy

| Signal | Quilt / vendor patches / inrepo series | Erigone overlay | Dual-file custom | Otto ledger | Git overlay branch |
| --- | --- | --- | --- | --- | --- |
| Unmodified | Path absent from patches | Path untouched by patch + overlay | No sibling file | Minimal / empty log | No overlay commits on path |
| Light | Small ordered patches | Thin wiring patch + new overlay files | Baseline hook + small custom | Moderate + few file entries | Few small commits |
| Heavy | Large replace-like patches | Prefer duplicate in overlay; avoid huge patches | Override sibling / full replace module | Heavy + high risk | Large commit surface |

**Mechanical vs editorial:** patch/overlay/dual-file can *derive* class from artifacts; ledgers are *editorial* and go stale unless process-enforced (otto: update ledger in the same commit as divergence).[^otto-sync]

---

## Fit notes specific to a Svelte builder + possible registry rewrite

Assumptions from map [#1](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/1): embeddable composer; host-registerable blocks; schema I/O; live preview seam; not upstream SPA chrome as product; upstream relationship still open.

Implications from this survey (still **not** a choice):

1. **Deep registry rewrite** behaves like erigone’s “new code should be overlay” rule: putting the rewrite only in patches against upstream registry files maximizes sync breakage.
2. **Thin upstream touch** (register hooks, export maps, one wiring call site) is what every successful patch-series practice optimizes for.
3. **npm packaging** makes `inrepo`-class or OpenTitan/Bender-class vendor+patches more native than Debian quilt alone; quilt ideas (ordered series, no fuzz) still transfer.
4. **Hard fork of a whole monorepo package** without a ledger recreates otto’s failure mode: nobody knows what is safe to cherry-pick.
5. **Host-registration API** can absorb some dual-file/`*.custom.*` lessons (stable extension points) *if* the public seam is designed for it — that is a product decision for later tickets, not this one.

---

## Criteria for a later grilling ticket

Use these as grill axes (questions, not answers). A choice should survive questions in every applicable row.

### A. Relationship & baseline

1. Is the artifact a **published package that vendors upstream**, a **GitHub fork**, or a **build-time overlay** that never ships upstream sources?
2. What is the **single pin** (git SHA / tag / lock file) that defines “upstream at last sync”?
3. Who owns the duty to **record baseline + last sync date** in-repo?

### B. Divergence taxonomy

4. How is a path classified as unmodified / light / heavy — **derived from artifacts**, **declared in a ledger**, or both?
5. What is the **maximum allowed patch surface** (files, LOC, or % of upstream package) before a change must move to overlay/replace?
6. Are **generated** transforms (import rewiring, branding) forbidden from entering the patch series? (inrepo separates rewiring from patches.[^inrepo])

### C. Re-apply mechanics

7. On upstream bump, is the operation **rebase series**, **re-apply quilt**, **rebase overlay branch**, **cherry-pick with ledger**, or **regenerate dual-file baselines**?
8. What happens on conflict — **hard fail CI**, **3-way auto-refresh**, **human continue/abort**, **skip + ledger note**?
9. Multi-version support: one pin, or **per-ref patch dirs** (erigone)?

### D. Registry rewrite specifically

10. Does the rewrite live as **new overlay modules**, **forked copies of upstream files**, or **in-place patches**?
11. Which upstream files remain **wiring-only** (must stay thin)?
12. Can host apps extend blocks without forking (dual-file / register API), and does that reduce Heavy surface?

### E. Operability

13. Can a new maintainer answer “what did we change?” from **committed files alone** in &lt;15 minutes?
14. Does CI **verify** recipe ≡ tree (inrepo verify / erigone check-patches / patch apply)?
15. Is upstream contribution path defined (refresh-patches / clean worktree PR) so fixes are not trapped forever in the fork?

### F. Failure budget

16. Acceptable sync cadence (every release / monthly / opportunistic)?
17. Acceptable manual conflict budget per sync (hours)?
18. Kill criteria: if Heavy paths exceed X or sync fails Y times, revisit relationship (wrapper / wait / greenfield)?

---

## Sources (primary)

[^quilt-man]: quilt(1) — Debian manpage: series file, push/pop stack, patch options. https://manpages.debian.org/unstable/quilt/quilt.1.en.html  
[^dpkg-source]: dpkg-source(1) — `3.0 (quilt)` extraction applies `debian/patches/series` with **no fuzz**. https://manpages.debian.org/jessie/dpkg-dev/dpkg-source.1.en.html  
[^debsrc30]: Debian Wiki — Projects/DebSrc3.0: quilt-compatible patch queue; `dpkg-source` applies/updates series; avoid leftover quilt rules in `debian/rules`. https://wiki.debian.org/Projects/DebSrc3.0  
[^opentitan-vendor]: OpenTitan — “Vendoring in Hardware” (`util/vendor.py`): lock files, `patch_dir`, `git apply -p1`, `patch_repo` + `--refresh-patches`, upstream-first policy. https://opentitan.org/book/doc/contributing/hw/vendor.html  
[^bender-vendor]: PULP Bender docs — Vendor workflow (`init` / `diff` / `patch`), inspired by OpenTitan `vendor.py`. https://pulp-platform.github.io/bender/workflow/vendor.html  
[^inrepo]: inthhq/inrepo README — pin + series (`git am --3way`), snapshot legacy format, `sync` / `update` / `verify`, conflict continue/abort. https://github.com/inthhq/inrepo  
[^erigone]: ethpandaops/erigone README — overlay + per-ref thin patches, save-patch, CI auto-refresh, “minimize patch surface” tip. https://github.com/ethpandaops/erigone  
[^erigone-commit]: ethpandaops/erigone — example of preferring duplicated overlay logic over widening patch surface (gas-sim intrinsic). https://github.com/ethpandaops/erigone/commit/9aeb9a90eb47aa2db533f153ed2b90873b851c67  
[^otto-sync]: cmetech/otto-cli `docs/UPSTREAM-SYNC.md` — fork baseline, package divergence table, file patch log, cherry-pick/port/skip workflow. https://github.com/cmetech/otto-cli/blob/main/docs/UPSTREAM-SYNC.md  
[^otto-pipeline]: cmetech/otto-cli `docs/UPSTREAM-PIPELINE.md` — operational companion to the ledger. https://github.com/cmetech/otto-cli/blob/main/docs/UPSTREAM-PIPELINE.md  
[^ampless-custom]: heavymoons/ampless — `*.custom.ts` extension points protected across upgrade. https://github.com/heavymoons/ampless/commit/31925fd3df2a3e0f8c6e44df22f5d5059b32070b  
[^clowder-local]: zts212653/clowder-ai — `.local` / `.local-override` overlay convention. https://github.com/zts212653/clowder-ai/commit/3230bf30c0af81e5d27a5f932fda94d7b7546c12  
[^pxt-custom]: Microsoft PXT — `custom.ts` for custom blocks (separate from `main.ts`). https://github.com/microsoft/pxt/blob/master/common-docs/blocks/custom.md  
[^routa-overlay]: Routa developer guide — Local Overlay And Upstream Sync (branch model, rebase flow, agent checklist). https://phodal.github.io/routa/developer-guide/local-overlay-sync  
[^git-vendor-replay]: tim-janik/git-vendor-replay — pristine vendor import commits + replay downstream patches. https://github.com/tim-janik/git-vendor-replay  
[^yarn-patch]: Yarn — Patch protocol + `yarn patch` / `patch-commit`. https://yarnpkg.com/protocol/patch  
[^pnpm-patch]: pnpm — `pnpm patch` / `patchedDependencies`, failure and unused-patch controls. https://pnpm.io/10.x/cli/patch  

---

## Explicit non-deliverables

- No ranking or recommendation of a single practice for this repo.
- No implementation of sync tooling, ledger files, or overlay layout.
- Upstream relationship choice (fork vs wrapper vs wait vs greenfield) remains deferred to later research/grilling per map [#1](https://github.com/CLOSERPROJECT/sjsf-form-builder/issues/1).
