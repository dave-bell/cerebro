# Changelog

All notable changes to this package are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this package adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Consumers declare this package as a bare dependency (`dave-bell/cerebro`), which
resolves to the tip of `main`. To take a new release, run `apm update`.

## [0.3.2] - 2026-10-29

### Changed

- **`durable-record` skill: when a decision needs an ADR.**
  - Policy choices now count as ADR-worthy: what a gate checks, ignores or tolerates; a
    security or compatibility trade-off; a convention adopted.
  - In projects whose `decisions-log.md` is an index of ADRs, every row must link an ADR.
    A choice too small for an ADR is too small for the log; mention it in the hand-back
    instead.
  - Read the log's header and Quick Reference first; they override the skill's defaults.
    Logs that say "one row per decision, no prose sections" get only the row, with
    everything else in the ADR.

## [0.3.1] - 2026-10-29

### Fixed

- **Agents work under Claude Code.** The `tools:` map (OpenCode's format) was deployed
  verbatim to `.claude/agents/`, where Claude Code expects a comma-separated string or a
  YAML list. Subagents launched without usable tools and wrote tool calls out as text.
  `tools` is removed from all five agents. Restrictions are now expressed with Claude
  Code's `disallowedTools` and OpenCode's `permission`.

### Changed

- **Tool restrictions are now enforced in OpenCode.** The old `tools:` map used capitalised
  names (`Read`, `Bash`, …) that never matched OpenCode's lowercase tools, so agents were
  unrestricted.
  - `context-indexer` and `context-retriever` are read-only (read, grep, glob and list only).
  - `context-organizer` has no shell or web access, and can call only `context-indexer`.
  - `context-manager` has no web access, and can call only `context-indexer`.
  - `external-lookup` can't call other agents.
  - All agents are now `mode: subagent`.
- **Read-only is advisory in Claude Code.** On macOS and Linux, Claude Code provides
  Glob and Grep to a subagent only through a `tools` list, which OpenCode rejects. So the
  read-only agents keep Bash for `find`/`grep`, and their instructions forbid any
  non-read-only use. Agent instructions no longer name harness-specific tools.

### Notes

- **OpenCode delegation.** OpenCode's default `subagent_depth` is 1, so
  `context-organizer` can't call `context-indexer` there. It falls back to its own
  discovery. Set `"subagent_depth": 2` in `opencode.json` to enable the delegation.
- **Headless Claude Code runs can't write under `.claude/`.** Claude Code treats
  `.claude/` as sensitive, so writes to `.claude/context/` need interactive approval. For
  automated runs, use a top-level `context/` root or set `contextRoot` in `.oac.json`.

## [0.3.0] - 2026-10-29

### Added

- **`durable-record` skill** A new skill to manage the context within a code change.

## [0.2.1] - 2026-08-07

### Fixed

- **`context-indexer` model selection.** The agent previously called out a model that wasn't valid.

## [0.2.0] - 2026-08-07

### Added

- **Root-level `navigation.md` template** (`.apm/skills/context-manager/assets/navigation.md`).
  This is the root nav for `{context_root}` itself, sibling to the
  `project-intelligence/` and `customer-intelligence/` domain folders. It was
  previously missing even though `context-root-discovery.md` requires it to
  detect an existing context root without `.oac.json`, and
  `context-indexer.agent.md` requires it to traverse into domain navs.
- `add-context` Stage 4 now creates `{context_root}/navigation.md` on first
  scaffold, and updates it in place when a second domain is added (existing
  domain rows are left untouched).
- `add-context` completeness check now verifies the root nav lists every domain
  folder present on disk.

### Fixed

- **`add-context` Stage 2 self-heals pre-existing context trees.** It now checks
  for a missing root nav *before* the domain-marker check and repairs scaffolds
  that have a domain folder but no `navigation.md`. Previously, any context tree
  created under 0.1.0 was permanently missing its root nav with no repair path.
  See "Upgrading" below — this writes to your existing context root.
- Dropped the orphaned `index.md` reference from the "Writing a New Context Root"
  scaffold instructions in `context-root-discovery.md`. Only `navigation.md` is
  ever read by any agent or reference file.

### Changed

- **Package renamed `context-agents` -> `cerebro`.** No consumer action beyond
  `apm update`. The `dave-bell/cerebro` dependency slug, the
  `apm_modules/dave-bell/cerebro/` path, and every deployed primitive filename
  (the `context-manager` and `context7` skills, the five agents, and the
  `context` / `add-context` commands) are unchanged. Only the `name:` field
  recorded in your `apm.lock.yaml` changes. No reinstall required, and no
  orphaned files are produced.

### Upgrading from 0.1.0

`apm update` deploys the new files, but it does not touch your context tree.
The self-heal above runs the next time you invoke `/add-context`: if your
`{context_root}` has a domain folder but no root `navigation.md`, one will be
created. This is the only change in this release that writes to consumer data.

Note that 0.1.0 shipped as two distinct commits (see below), so the exact
delta you receive depends on which commit you were locked to.

## [0.1.0] - 2026-07-22

Initial release: context management agents, skills, and commands for AI coding
assistants — project standards discovery, MVI-compliant context generation, live
library doc lookup, and context lifecycle management across the
`software-development` and `labbit-configuration` domains.

### Unversioned changes released under 0.1.0

The following shipped in commit `c03852e` (2026-07-24) without a version bump,
so consumers locked to the initial commit `44e5435` will also pick these up when
they move to 0.2.0:

- `add-context` Stage 4 enumerates the exact six template files for both dev and
  config modes, rather than implying "fill the template set" — which allowed
  models to omit files or improvise alternate structures. Adds an explicit
  prohibition on inventing folders or using YAML frontmatter, plus a
  pre-completion verification gate.
- `context-organizer` frontmatter rule states that the HTML-comment format is
  mandatory and YAML is disallowed. Post-generation checklist gained a
  frontmatter format check and a scaffold-completeness check.

Together these fixed a gap where a fresh dev scaffold could silently omit
`business-domain.md`, `business-tech-bridge.md`, `decisions-log.md`, and
`living-notes.md` while producing only `technical-domain.md` with non-standard
YAML frontmatter.
