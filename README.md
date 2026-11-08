# agentpack: design and research

This repository holds the design work for **agentpack**, a CLI concept for moving an AI coding agent's configuration between agents and machines. A bundle would carry skills, MCP server definitions, hooks, slash commands and subagents, with secrets referenced by URI instead of inlined.

Nothing here is implemented. There is no source code, no binary, no tests and no released tool. This is a specification and a research record. [STATUS.md](STATUS.md) states the exact line between what is written down and what exists.

## Contents

| Path | What it is |
|---|---|
| [docs/prd-agentpack.md](docs/prd-agentpack.md) | PRD v1.0: problem, 5 goals, 5 user stories, 17 functional requirements, non-functional requirements, architecture, 10-week roadmap, out-of-scope list, risks |
| [docs/prd-agentpack-scope-gaps.md](docs/prd-agentpack-scope-gaps.md) | Addendum with 6 more requirements: global and per-project CLAUDE.md, auto-memory, keybindings, runtime environment, skill registries |
| [docs/prd-agentpack-validation-report.md](docs/prd-agentpack-validation-report.md) | Self-validation of the PRD against a 13-check checklist, scored 57/57 |
| [docs/tasks-preview-agentpack.md](docs/tasks-preview-agentpack.md) | 47-task Phase 1 breakdown with acceptance criteria, dependencies and estimates |
| [docs/agentpack-status.json](docs/agentpack-status.json) | Task ledger with per-item status |
| [research/skill-exporter-research-brief.md](research/skill-exporter-research-brief.md) | Competitor map, gap analysis, build-vs-buy options, red-team review |
| [research/skill-export-log-2026-06-15.md](research/skill-export-log-2026-06-15.md) | Log of a manual Windows-to-WSL config copy used as a proof of concept |

## The idea in one paragraph

Export reads an agent's config directory and writes a portable bundle: deterministic tar plus zstd, a YAML manifest, and artifacts addressed by SHA-256. Secret values never enter the bundle. A config value points at `secret://env/NAME` or `secret://keychain/service/account` instead, and the install side resolves it against the local machine's backend. The bundle can be signed with minisign/ed25519. Install translates the canonical representation into the target agent's format: Claude Code JSON, Codex TOML, opencode JSONC or Cursor JSON. A separate `wsl-sync` command handles the Windows and WSL split, including path translation and line endings.

## Reading order

1. The research brief for the market picture and the gap the design targets.
2. The PRD for the full specification.
3. The scope-gaps addendum for the requirements added after the first PRD draft.
4. The task breakdown for the planned build order.
5. The POC log for the one behaviour that was exercised by hand.

## Publication edits

The material was written in June 2026. Before publication:

- Personal machine paths in the POC log were replaced with `<user>` and `<project>` placeholders.
- Log timestamps were converted from the original capture to IST (UTC+05:30).
- A personal email address was removed from the PRD author fields.
- Relative links between the documents were repaired.

Technical content is otherwise unchanged.

## License

The specification and research text in this repository are licensed under Creative Commons Attribution 4.0 International. See [LICENSE](LICENSE). No implementation is present. The task plan suggests Apache-2.0 for the future code, which is a separate decision.
