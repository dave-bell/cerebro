---
name: durable-record
description: Use when a change is implemented and verified and its durable record must be written into the project context before the change is handed off for review — decisions (ADR and/or decisions-log), requirement traceability (business-tech-bridge), and open questions, debt and active work (living-notes). Triggers on "write the ADR", "record the decision", "update decisions-log", "update living notes", "update the bridge", "write the durable record", and on finishing any ticket, issue or task whose work settled a decision or moved something the project context tracks.
---

# durable-record

Turn a change's settled decisions into the context artifacts a future reader will actually find, and ship them **with the change itself**, in the same branch and the same review. This is the write-side counterpart to the `context-indexer` agent: it reads the context before work starts, and this skill updates it when work ends.

Works the same under any harness (Claude Code, opencode, Codex, an automated coding agent). Nothing here assumes a particular command, a worktree, a git host CLI, or permission to commit.

**Announce at start:** "Writing the durable record for this change."

## The Rule: The Record Ships With the Change

The record is part of the deliverable, not a follow-up:

1. **Same change, same review.** Context updates land in the same branch or pull request as the code that justifies them, and are reviewed alongside it.
2. **Read plans literally.** If the task, issue or plan names a context file or an ADR as a deliverable, it is one.
3. **No after-merge write-ups.** A record written later, from memory, loses exactly the parts nobody can reconstruct: the rejected alternatives and the surprises found during implementation.

## Step 0: Preconditions

Do not start until these hold:

| Check | Why |
|---|---|
| The implementation is **complete and verified** (the project's checks pass) | The record describes what the code does, not what it was going to do |
| You have read the **task and its discussion** (issue comments, review notes, linked decisions), as far as they're available to you | The discussion often amends the original plan; the record must reflect what was actually decided |
| `{context_root}` is resolved | See the `context-manager` skill: `references/context-root-discovery.md`. Resolve once per session. |
| You have read the **project's own conventions** | Project instructions (`AGENTS.md`, `CLAUDE.md` or equivalent) and workflow guides under `{context_root}` may set the ADR location, numbering, model ADR, ID formats and commit style. **Project conventions override the defaults in this skill.** |

If `{context_root}` doesn't resolve, there is no context system to record into: say so and stop. Don't scaffold one as a side effect (that's the `add-context` prompt's job, with approval).

## Step 1: Decide What Applies

Not every change earns every artifact. Decide deliberately, and **say which you skipped and why**; silence reads as an oversight.

| Artifact | Write it when | Skip it when |
|---|---|---|
| **ADR** (only if the project keeps ADRs, see Step 2) | The change settled an architectural choice, or reversed one | Pure mechanics with no choice in it: a rename, a dependency bump, a typo |
| `{context_root}/project-intelligence/decisions-log.md` | A decision was made (with or without an ADR) | No decision: nothing was chosen between alternatives |
| `{context_root}/project-intelligence/business-tech-bridge.md` | Requirement or feature traceability changed: new requirements covered, a mapping now points at real code or tests, a trade-off was made | The change touches no business ↔ tech mapping |
| `{context_root}/project-intelligence/living-notes.md` | The work opened, closed or narrowed an open question, added or paid down technical debt, revealed a gotcha, or started or finished an active project | Nothing on that file's lists moved |
| `{context_root}/customer-intelligence/customization-log.md` | The change is customer configuration work and the project keeps customer-intelligence | Not customer configuration work |

If **nothing** applies, make no context changes and report that in the hand-back. An empty or cosmetic record is noise.

## Step 2: The ADR (Only If the Project Keeps ADRs)

**Find out whether the project uses ADRs**, in this order:
1. The project's instructions or a workflow guide under `{context_root}` name an ADR location.
2. An existing ADR directory: `docs/adr/`, `docs/decisions/`, `doc/adr/`, `adr/`, `architecture/decisions/`.
3. Neither → **the project doesn't use ADRs.** Put the full decision in `decisions-log.md` using its template (Step 3), and don't create an ADR directory.

When the project does keep ADRs:
1. **Number from the directory, not from the task text.** List the ADR directory and take the highest number plus one. Task text names a number when it's written and drifts. If they disagree, follow the directory and note the divergence in the hand-back.
2. **Match a neighbour.** Read the most recent ADR, or the model ADR the project names, and follow its structure, title format and filename pattern. Typical shape: `# ADR-NNNN: Title`, `## Status`, `## Context`, `## Decision`, `## Consequences`.
3. **Status is accepted on merge.** Write `Accepted — YYYY-MM-DD` with the deciding issue or task linked, unless the project's convention differs. Not `Proposed`: merging the change is the acceptance, and a `Proposed` status needs a later flip, which is exactly the dangling follow-up this skill exists to remove.
4. **Never hedge on its own status.** Don't write that the ADR "is not yet live". It ships with the code it describes.
5. **Capture what can't be reconstructed later:** the alternatives rejected and why, and any correction the implementation forced on the plan.

## Step 3: `decisions-log.md`

Follow the structure **this project's copy** documents. Read the file first:

- **With ADRs:** the log is the index and summary. If the file has an index table, add a row. Add an entry following the file's documented template, but **summarise in two to four sentences and link the ADR**. Rationale, alternatives and consequences live in the ADR, in exactly one place.
- **Without ADRs:** the log entry is the record. Use the file's **Decision Template** in full: Date, Status, Owner, Context, Decision, Rationale, Alternatives Considered, Impact, Related.

Either way, link the related issue, task or pull request, and the code (see the `context-manager` skill: `references/engine/standards/codebase-references.md`).

## Step 4: `business-tech-bridge.md` and `living-notes.md`

Only where Step 1 said they apply. **Edit in place, following each file's existing entry shape.**

- `business-tech-bridge.md`: update the mapping the change affected (feature or requirement → code and tests, trade-offs). Point at real paths.
- `living-notes.md`: add to or update the right list: Technical Debt, Open Questions, Known Issues, Insights & Lessons Learned, Patterns & Conventions, Active Projects. **Closing an item means moving it to the Archive section with its resolution, not deleting it.**

## Step 5: Context-File Hygiene

- **Bump the header** of every context file you touched: `Version:` (minor bump) and `Updated:` (today) in the `<!-- Context: … -->` comment on line 1. See the `context-manager` skill: `references/engine/standards/frontmatter.md`.
- **Respect MVI size limits** (`references/engine/standards/mvi.md`). If an entry would push a file over its limit, load the `context-manager` skill and compress (split by concern, move detail into a sibling reference file) rather than letting it sprawl.
- **Keep navigation true.** If you created a context file, add it to the nearest `navigation.md`.

## Step 6: Don't Edit Read-Only Sources

Some material the record relates to may be **read-only for you**: requirements in a wiki or another repository (often mounted as a git submodule), vendored documentation, generated files. Don't edit them, even if you can. Instead, write the exact change they need into the hand-back under **Requirements impact**, so a human can apply it.

## Step 7: Commit, If Your Harness Lets You

Choose the case that applies:

- **You commit, and the project wants records separate** (check its conventions): make **one record commit after the implementation commits**, containing only the record files, with a subject that names the artifacts written. For example: `record the secrets store adapter decision: adr-0007, decisions-log, living-notes.` Follow the project's commit style; with no convention, don't put a ticket ID in the subject (the pull request carries the link).
- **You commit, with no convention for separate commits:** include the record in the change's commit(s) as usual.
- **You aren't permitted to commit, or the harness commits for you** (common for automated agents): **don't commit and don't work around it.** Leave the record files modified in the working tree, and list them in the hand-back so the harness and the reviewer can see them.

## Step 8: Hand Back

Report, in this shape, so it can be dropped straight into a pull request description:

```markdown
## Context updates
- `docs/adr/0012-hex-audit-in-precommit.md`: new ADR (ADR-0012)
- `{context_root}/project-intelligence/decisions-log.md`: index row + summary
- `{context_root}/project-intelligence/living-notes.md`: closed "dependency audit gap" (moved to Archive)
- Skipped `business-tech-bridge.md`: no requirement mapping changed

## Requirements impact
None   (or: the exact text to change in which read-only source, and why)
```

Also mention any place where the task text disagreed with what you found (ADR numbering, a plan the discussion amended).

## If the Change Already Merged Without Its Record

A missed record is still worth writing. Write it as a standalone change on the default branch, and say plainly that it repairs a missed step and isn't the normal flow.

## Related

- **Finding context before work starts** → `context-indexer` agent
- **Resolving `{context_root}`** → `context-manager` skill: `references/context-root-discovery.md`
- **What project-intelligence captures, and how to manage it** → `context-manager` skill: `references/domains/software-development/project-intelligence.md` and `project-intelligence-management.md`
- **Size limits, headers, linking to code** → `context-manager` skill: `references/engine/standards/mvi.md`, `frontmatter.md`, `codebase-references.md`
- **Compressing an oversized context file** → `context-manager` skill, COMPRESS operation
