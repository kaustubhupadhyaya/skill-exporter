# Status

Design written between 2026-06-14 and 2026-06-18. Published 2026-11-08 IST.

This is a specification repository. No application code exists.

## Specified

- Bundle format: deterministic tar plus zstd, `manifest.yaml`, schema version 1, optional minisign/ed25519 signature (REQ-001, REQ-006).
- Commands: `export`, `install`, `inspect`, `lint`, `verify`, `sign`, `keygen`, `secrets`, `wsl-sync`, `diff`, plus project-scoped export and install in the addendum.
- Artifact coverage: skills, slash commands, subagents, MCP servers, hooks (Phase 2), permissions (Phase 2), global and per-project CLAUDE.md, auto-memory, keybindings.
- Cross-format MCP translation: Claude JSON, Codex TOML, opencode JSONC, Cursor JSON (REQ-004).
- Secrets: `secret://` URI scheme with env and keychain backends in the MVP, age/sops/1Password in Phase 2, and mandatory inlined-secret detection on export (REQ-005, REQ-007).
- Windows and WSL bridge: path translation, CRLF/LF conversion, collision policies (REQ-008).
- Lint rules and remote bundle fetch (REQ-009, REQ-013).
- Roadmap: Phase 1 is 47 tasks, about 184 unbuffered hours or 230 buffered hours. The scope addendum raises this to 196 or 242.
- Research: competitor map, build-vs-buy analysis, red-team risks.

## Not built

- No source code of any kind, no `go.mod`, no `schemas/manifest.v1.json`, no CI, no tests, no releases.
- No `agentpack` CLI, so no export, install, inspect, lint, sign, verify or secrets commands.
- No bundle writer or reader, no installers for any agent, no MCP translators.
- No secret backends and no inlined-secret scanner.
- No WSL sync automation. A person followed the procedure by hand once, on one Windows and WSL pair, and logged it.
- No reference bundles and no published schema file.

## What the status ledger means

[docs/agentpack-status.json](docs/agentpack-status.json) marks five preparation items done and T-030 to T-033 partial. The partial items are behaviours exercised manually in the POC, not code. Every implementation task (T-001 to T-047) is otherwise pending. The six addendum tasks (G-018 to G-023) are pending except G-022, which is deliberately skipped.

## Limitations

- The 57/57 validation score is a self-check against a local checklist. The validation script was not installed, so the score is not an independent review.
- Estimates are planning numbers, not measured build times.
- The scope-gaps addendum has not been merged into the PRD.
- Competitor star counts were approximate when written.
- The POC covered one machine, one distribution and one day.
