# cerebro

Universal context management agents, skills, and prompts for AI coding assistants — extracted and generalized from [OpenAgentsControl](https://github.com/darrenhinde/OpenAgentsControl)'s context system, without its top-level orchestration layer.

Distributed as an [APM](https://github.com/microsoft/apm) (Agent Package Manager) package. Author once, deploy to Claude Code, opencode, Codex, and other harnesses via `apm compile`.

This package ships two content domains out of the box: **software-development** (building the product) and **labbit-configuration** (consultants configuring **Labbit** — an enterprise solution — for a specific customer, via JSONC config docs and BPMN process files). Rename or add domains as needed for your own product; the engine and agents are domain-agnostic.

## What's Included

**Agents** (`.apm/agents/`)
- `context-indexer` — discovers and ranks relevant context files before implementation (read-only)
- `context-retriever` — generic context search for repos with no formal context system (read-only)
- `external-lookup` — fetches live library/framework docs via Context7, persists to `.tmp/external-context/`
- `context-organizer` — generates MVI-compliant context files (calls `context-indexer` first, always)
- `context-manager` — catalogs, validates, and maintains context structure and lifecycle

**Skills** (`.apm/skills/`)
- `context-manager` — the operational engine and both content domains, in three tiers:
  - `references/engine/` — domain-agnostic rules shared by both domains: MVI, frontmatter, structure, codebase references, operations (harvest, extract, organize, update, migrate, error)
  - `references/domains/software-development/` — `project-intelligence.md` (standard) + `project-intelligence-management.md` (management guide) + `standards/` (7 language-agnostic coding standards: code quality, testing, docs, security, analysis, clean code, API design), for building Labbit itself
  - `references/domains/labbit-configuration/` — `customer-intelligence.md` (standard) + `customer-intelligence-management.md` (management guide) + `standards/` (JSONC config patterns, BPMN modeling, customer config governance), for consultants configuring Labbit for a specific customer
  - `assets/project-intelligence/` and `assets/customer-intelligence/` — the template sets copied into a consumer project's `{context_root}` by the `add-context` prompt
- `context7` — low-level Context7 API fetch mechanism for live library docs

**Prompts** (`.apm/prompts/`) — become `/add-context` and `/context` commands on every target that supports slash commands
- `add-context` — wizard to create/update project-intelligence or customer-intelligence context
- `context` — run a context maintenance operation (discover, harvest, extract, organize, cleanup, etc.)

## Two Domains, One System

| Mode | Use when | Context lives at | Standard + management guide + coding/config standards |
|---|---|---|---|
| **dev** | Building/extending Labbit itself | `{context_root}/project-intelligence/` | `project-intelligence.md` + `project-intelligence-management.md` + `standards/` (code quality, testing, docs, security, analysis, clean code, API design) |
| **config** | Configuring Labbit for a named customer | `{context_root}/customer-intelligence/` | `customer-intelligence.md` + `customer-intelligence-management.md` + `standards/` (JSONC config, BPMN modeling, customer config governance) |

Both domains have the same shape and are governed by the same shared engine (MVI, frontmatter, structure standards), discovered the same way. Each domain's management guide references the engine for generic mechanics (versioning, deprecation, size limits) and only documents what's specific to that domain — no duplication between them, and no duplication between the two domains' `standards/` directories either. A project can use either domain or both.

## Tool-Agnostic by Design

Every agent and reference file resolves its context root dynamically (see `context-root-discovery.md` in the `context-manager` skill) rather than hardcoding a tool-specific directory. This is what makes the same package work under Claude Code (`.claude/context/`), opencode (`.opencode/context/`), Codex (`.codex/context/`), or a plain `context/` folder.

## Install

```bash
apm install dave-bell/cerebro
```

This deploys agents, skills, and prompts to every detected/configured harness (see `apm.yml` `targets:`). To target specific harnesses only:

```bash
apm compile -t claude
apm compile -t codex
apm compile -t opencode
```

## Getting Started

1. Install the package (above).
2. Run `/add-context` (or the equivalent prompt invocation on your harness) and choose `dev` or `config` mode.
3. Answer the short wizard — 5-6 questions, ~5 minutes.
4. Agents now discover and follow your patterns automatically via `context-indexer`.

## Known Cross-Tool Limitations

- **Codex** does not carry fine-grained `tools:` restrictions from agent frontmatter (APM limitation) — agents run with whatever tools the Codex harness grants by default.
- **Codex and OpenCode** have no slash-command/hooks primitive for some flows — the `/add-context` and `/context` prompts still work as commands on Claude, Cursor, opencode, and Gemini; Codex users can ask the agent to run the same workflow conversationally.
- Fine-grained path-scoped permissions (e.g. "only write files under `context/`") are enforced by instruction in each agent's prompt body, not by the tool-permission system — APM's cross-tool `tools:` field is coarse (whole-tool allow/deny), unlike OpenCode-native `permission` blocks.

## Provenance

The subagents, standards, and operational patterns here originate from OpenAgentsControl's context system (`ContextScout`, `ExternalScout`, `ContextOrganizer`, `ContextManager`, `Context Retriever`, the `core/context-system/` engine, and the language-agnostic `core/standards/` + `development/principles/` coding standards). This package intentionally excludes OAC's top-level orchestration agents (`OpenAgent`, `OpenCoder`, `SystemBuilder`) and its OAC-repository-specific context domains. Several broken cross-references and two workflow docs with missing implementation scripts (`session-context-pattern.md`, `lightweight-context-handoff.md`) were fixed or dropped during the port. The ported coding standards had OAC agent-name metadata (`**Used by**: opencoder`, etc.) removed since those agents aren't part of this package.
