# Cerebro Gap Report: Missing Root `navigation.md` in `add-context` Scaffold

**Filed against**: `dave-bell/cerebro` (pinned commit `c03852e803e720639ed191a070f0c23d4cf3633d` in this consumer)
**Found in**: `bedrock_chat` project, while wiring `context-indexer` into `CLAUDE.md` planning/build workflow.
**Severity**: High — breaks the documented discovery contract for any project that doesn't lean on `.oac.json`, and silently leaves already-scaffolded projects broken forever (see D4).

---

## Summary

`add-context` scaffolds only the domain-specific file set (e.g. `project-intelligence/`'s 6 files) at `{context_root}`. It never creates a root-level `{context_root}/navigation.md`. But `context-root-discovery.md` (the resolution protocol every agent follows) and `context-indexer.agent.md` (the discovery workflow) both require that root nav to exist — one to *detect* an existing context root without `.oac.json`, the other to *traverse* into domain navs. There's a three-way contradiction inside the package's own docs, plus an orphaned `index.md` reference, plus no repair path for context trees already scaffolded under the current (broken) instructions.

---

## D1: No root navigation template exists

`assets/` only contains:
```
.apm/skills/context-manager/assets/
├── customer-intelligence/   (6 files, no root nav)
└── project-intelligence/    (6 files, no root nav)
```

Neither subfolder's file set includes a root-level `navigation.md` for `{context_root}` itself — only `project-intelligence/navigation.md` / `customer-intelligence/navigation.md`, which are domain-level, not root-level.

But `context-root-discovery.md:37-43` ("Writing a New Context Root") explicitly requires one:

> 2. Create `{context_root}` = `<detected-tool-dir>/context/` if a tool dir exists, else plain `context/` at the project root.
> 3. **Scaffold `navigation.md`, `index.md`, and either `project-intelligence/` ... or `customer-intelligence/`** ...

There is no template to scaffold from.

**Fix**: Add `assets/navigation.md` — a root-level nav template, sibling to (not inside) the domain folders. Suggested content (adapted from this project's now-hand-written stopgap, and following the package's own "Category Navigation Template" in `references/engine/guides/navigation-templates.md:9-51`):

```markdown
<!-- Context: core/navigation | Priority: critical | Version: 1.0 | Updated: {date} -->

# Context Root Navigation

**Purpose**: Entry point for the context system. Route to the right domain before reading anything else.

---

## Structure

```
{context_root}/
├── navigation.md              # This file - start here
└── {project-intelligence and/or customer-intelligence}/
```

---

## Quick Routes

| Task | Path |
|------|------|
| **Building/extending the product (software dev)** | `project-intelligence/navigation.md` |
| **Configuring a customer solution (labbit-config)** | `customer-intelligence/navigation.md` |

*(Omit whichever domain row doesn't apply yet; add it later if `/add-context` is run again with the other mode.)*

---

## By Domain

**project-intelligence** → business/technical context for building the product itself.
**customer-intelligence** → customer profile, config, and process-map context for a named implementation.

---

## Related Context

- **project-intelligence** → `project-intelligence/navigation.md` (if present)
- **customer-intelligence** → `customer-intelligence/navigation.md` (if present)
```

Placeholders (`{context_root}`, `{date}`, which domain rows are present) get filled by `context-organizer` the same way it fills the existing domain templates.

---

## D2: `add-context` Stage 4 forbids creating it, and its pre-flight check doesn't look for it

`add-context.prompt.md:59-77` (Stage 4: Generate) says:

> **Instantiate the complete template set for the chosen mode.** Copy every file from the mode's `assets/` folder into the target folder... **Do NOT invent alternate folders... or alternate frontmatter styles. The template set defines the exact files and format for both modes.**
>
> ...
>
> **Before reporting done on a fresh scaffold**, list the target folder and confirm every template file for the chosen mode is present...

This instructs the agent to write *only* the domain's 6 files, and its own "did I finish" check only verifies those 6 — the root nav is invisible to both the instruction and the verification.

**Fix**: Update Stage 4 to:
1. Copy `assets/navigation.md` to `{context_root}/navigation.md` **if it doesn't already exist** (first domain added to this root), or **update it in place** to add a row for the newly-added domain (second domain added to an existing root — don't overwrite the existing domain's row).
2. Extend the pre-flight "before reporting done" check to also confirm `{context_root}/navigation.md` exists and lists every domain folder actually present on disk.

---

## D3: Orphaned `index.md` reference

`context-root-discovery.md:43` mentions scaffolding `index.md` alongside `navigation.md`. This is the **only** occurrence of `index.md` anywhere in the package — grepped the full `.apm/` tree, zero other hits. No agent reads it, no other reference file mentions it, `add-context` doesn't generate it, and the package's own established pattern (confirmed by every other nav-related file) is `navigation.md`, not `index.md`.

**Fix**: Delete `index.md` from the `context-root-discovery.md:43` sentence. It should read:

> 3. Scaffold `navigation.md` and either `project-intelligence/` (software-development domain) or `customer-intelligence/` (labbit-configuration domain) — see the `add-context` prompt.

---

## D4: No repair path for context trees already scaffolded under the current instructions

Every project that ran `/add-context` before this fix lands has a domain folder but no root nav — and will silently stay that way forever, because Stage 2 ("Detect Existing Context") only checks for domain-specific marker files (`technical-domain.md` for dev, `customer-profile.md` for config — see `add-context.prompt.md:32`). If those exist, it treats the context tree as fully set up and routes straight to the review/update flow for that domain's files. It never inspects whether `{context_root}/navigation.md` itself exists.

Concretely: after this fix ships and a project re-runs `/add-context`, it will still be missing its root nav, because "exists" short-circuits before the new root-nav logic in D2 ever runs.

**Fix**: Add a structural-integrity check, run before the mode-specific Stage 2 branch:
1. Does `{context_root}/navigation.md` exist?
2. If not, and at least one domain folder exists → this is a pre-fix scaffold. Generate the root nav (per D1/D2) listing whatever domain folders are actually present, report it as a repair, then continue into the normal Stage 2 flow.
3. If not, and no domain folder exists → fresh scaffold, proceed as today.

This makes `/add-context action=update` on an existing (broken) project self-healing without requiring a `Replace all`.

---

## Downstream Effect: Discovery Fragility Without `.oac.json`

`context-root-discovery.md` resolution order is:
1. `.oac.json` / `apm.yml` `context.root` — explicit config
2. Glob for `navigation.md` in tool-native dirs — **uses root nav as the existence signal**
3. Global fallback
4. Nothing found → offer `add-context`

Because D1-D2 mean no scaffold ever produces that root `navigation.md`, **step 2 can never succeed** for any project relying on it instead of an explicit `.oac.json`. Every consumer either needs `.oac.json` written by hand (as happened in `bedrock_chat`), or discovery falls through to "nothing found" even though a domain folder is sitting right there. This is the practical blast radius of D1 — worth calling out explicitly in the PR/commit description so reviewers understand why D1 is High severity, not cosmetic.

---

## Verification After Fix

1. In a scratch project with no existing context, run `/add-context mode=dev`. Confirm `{context_root}/navigation.md` and `{context_root}/project-intelligence/navigation.md` both exist, and the root nav's Quick Routes table points correctly to the domain.
2. Run `/add-context mode=config` in the same project afterward. Confirm the root nav now lists both domains, and neither domain's existing files were touched.
3. In a project scaffolded *before* this fix (domain folder present, root nav absent), run `/add-context action=update`. Confirm the root nav gets created (repair path, D4) without prompting `Replace all` on the domain files.
4. Delete `.oac.json` in a fixed project and confirm `context-indexer` / `context-root-discovery` step 2 (glob for `navigation.md`) resolves the root correctly on its own.
5. `grep -rn "index.md"` the package tree — zero hits (D3).

---

## Consumer-Side Status (for reference)

`bedrock_chat` has already applied a hand-written stopgap at `.opencode/context/navigation.md` (git-tracked, since `.opencode/context/` is consumer data never regenerated by `apm compile`). Once this fix lands upstream and the pin is bumped, that file can be diffed against the new `context-organizer`-generated output and reconciled — it was modeled directly on the fix proposed in D1, so drift should be minimal to none.
