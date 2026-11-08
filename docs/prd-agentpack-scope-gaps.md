# agentpack — Scope Gaps vs Richer Problem Statement

**Date:** 2026-06-18
**Source:** [prd-agentpack.md](./prd-agentpack.md) v1.0 vs user's expanded problem statement
**Status:** Addendum — does not rewrite the original PRD; lists deltas + new REQs to fold in when the PRD slot frees up again.

---

## Coverage matrix

| Layer | Original PRD | Gap action |
|---|---|---|
| Skills (`~/.claude/skills/`) | REQ-001..003 | none |
| Hooks (PTU / PostToolUse / UPS) | REQ-010 (P2) | none |
| CLAUDE.md global | — | **NEW REQ-018** (P1) |
| CLAUDE.md per-project | — | **NEW REQ-019** (P1, extends REQ-002 scanner) |
| Auto-memory `~/.claude/projects/<slug>/memory/` | — | **NEW REQ-020** (P1) |
| MCP servers (`~/.mcp.json`) | REQ-004 | none |
| Permissions (settings.json) | REQ-012 (P2) | none |
| Keybindings (`~/.claude/keybindings.json`) | — | **NEW REQ-021** (P2) |
| Runtime env (venvs / NPM globals / shell utils) | — | **NEW REQ-022** (P3 / non-goal candidate) |
| Project `.claude/` subtrees (per-repo settings, symlinks, MD) | partial | **EXTEND REQ-002/003** |
| Third-party skill registries (`skills@latest`-style) | — | **NEW REQ-023** (P2) |

---

## New requirements (sketches, not full task breakdowns)

### REQ-018 — CLAUDE.md global sync (P1, ~3h)

Treat `~/.claude/CLAUDE.md` as a first-class artifact in the bundle. Single file, plain copy, no translation. Lint warns if it references absolute paths (`C:\…` / `/home/<user>/…`) — those break on import. Phase: MVP. Adds to REQ-002 scanner + REQ-003 installer.

### REQ-019 — Per-project CLAUDE.md + `.claude/` subtree (P1, ~4h)

Bundle a project's `.claude/` directory as a unit: `.claude/CLAUDE.md`, `.claude/settings.json`, `.claude/settings.local.json` (with secrets-detector running first), `.claude/skills/`, `.claude/agents/`, `.claude/commands/`. Treat symlinks: follow + record target, OR record as symlink + recreate on install (choose: follow, matches Layer A POC behavior). New CLI: `agentpack export-project <path>` and `agentpack install-project <bundle> --into <path>`.

### REQ-020 — Auto-memory directory sync (P1, ~5h)

`~/.claude/projects/<project-slug>/memory/` contains user / feedback / project / reference Markdown files written by Claude across sessions. Bundle this as-is with one rule: **never include without explicit `--include-memory` flag** (memory is private). Add lint rule to detect PII patterns (emails, full names, repo URLs in feedback type) and warn before bundle. Adds `memory:` to manifest artifacts schema.

### REQ-021 — Keybindings (P2, ~2h)

`~/.claude/keybindings.json` is small + single-file. Sync as part of settings layer in Phase 2 (REQ-012 expanded). Lint warns if any binding references a custom-path script that's not also in the bundle.

### REQ-022 — Runtime env (P3 or explicit non-goal)

User's hooks/skills often depend on `python -m foo`, `node bar`, or shell helpers installed in `~/bin/`. Three options:
- **(a)** Out of scope — document required deps in skill SKILL.md, user installs separately. Cheapest.
- **(b)** Lockfile emit — bundle includes `requirements.txt` / `package.json` / `Brewfile` fragments. Install side runs `pip install -r` / `npm i -g` etc. Risky (network at install time, version drift).
- **(c)** Container image — bundle is a Dockerfile or OCI image. Heavy. Defers to user's existing devcontainer setup.

**Recommendation:** ship as explicit non-goal in MVP, revisit in Phase 3 with option (a) baseline. Saves ~20h of scope creep.

### REQ-023 — Third-party skill registries (P2, ~4h)

`agentpack install skills://skills.dev/foo@latest` resolves a registry URL (skill-package format `<name>@<version>`) before bundling/installing. Two registries in scope: skills.dev (hypothetical, the `skills@latest` pattern in user's statement) and the official Claude Code Plugin marketplace (we're already adjacent). Skips a roll-our-own registry — defer.

---

## Phase placement summary

| New REQ | Phase | Hours | Rationale |
|---|---|---|---|
| REQ-018 (global CLAUDE.md) | P1 | 3 | Tiny scope, big symbolic value (your identity layer) |
| REQ-019 (project subtree) | P1 | 4 | Same trip through the scanner; cheap to add |
| REQ-020 (auto-memory) | P1 | 5 | The one we obviously missed; ship in MVP |
| REQ-021 (keybindings) | P2 | 2 | Adds to existing settings-layer work |
| REQ-022 (runtime env) | P3 / non-goal | 0 in MVP | Defer; document as non-goal |
| REQ-023 (skill registries) | P2 | 4 | Useful but not blocking |
| EXTEND REQ-002/003 (project) | P1 | (inside 019) | — |

**Phase 1 delta:** +12 hours raw, +15 buffered. Phase 1 total now ~242h buffered.
**Phase 2 delta:** +6 hours raw, +8 buffered.

---

## What to ship the original PRD with (when restored)

When `.taskmaster/docs/prd.md` slot is reused for agentpack again:
1. Restore `prd-agentpack.md` → `prd.md` (or keep as `prd-agentpack.md` if multi-project).
2. Append the 6 new REQs above to the **Functional Requirements** section using the same template.
3. Update the totals in **Implementation Roadmap → Effort Estimation**: raw 196h (was 184h), buffered 242h (was 230h).
4. Update **Out of Scope** to add REQ-022 (runtime env) as explicit non-goal in MVP with the (a)/(b)/(c) options noted as Phase 3 decision.
5. Update **Goal 1 (TTRC)** measurement fixture: include 1 CLAUDE.md, 5 memory files, 1 keybindings.json in the 30-artifact baseline.

No further action on prd-agentpack.md until that restore. This addendum is the source of truth in the meantime.
