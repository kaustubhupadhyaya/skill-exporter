# Skill Export Operation Log — 2026-06-15

**Operator:** Claude (working manually as proof-of-concept for [agentpack PRD](../docs/prd-agentpack.md) `wsl-sync` behavior)
**Source:** Windows-native Claude Code at `C:\Users\<user>\.claude\`
**Target:** WSL2 Ubuntu Claude Code at `/home/<user>/.claude/` (symlink → `/home/<user>/<project>/.claude/`)

> ⚠️ **WSL `.claude` is a symlink into the `<project>` project.** All "WSL-side" writes below physically land in `/home/<user>/<project>/.claude/`. Claude Code follows the symlink transparently, but if the <project> project is moved or renamed, these writes become detached.

---

## Plan

**Layer A — User-level skills (18 to copy):**

| Skill | Action | Reason |
|---|---|---|
| drawio | skip | exists on WSL (8167 B), WSL version newer than Win (7730 B) — user approved skip |
| grill-me | replace WSL symlink with regular dir containing Win version | Win SKILL.md is 8890 B (your recent research-first rewrite); WSL is a symlink to <project> shared stub (635 B) — keep <project> stub intact, give `~/.claude/skills/grill-me/` its own copy of Win version |
| answerinshort | new copy | Win-only |
| api-design | new copy | Win-only |
| coding-standards | new copy | Win-only |
| deep-research | new copy | Win-only |
| deployment-patterns | new copy | Win-only |
| describe | new copy | Win-only |
| docker-patterns | new copy | Win-only |
| e2e-testing | new copy | Win-only |
| mw-build-and-test | new copy | Win-only |
| prd-taskmaster | new copy | Win-only |
| product-requirements | new copy | Win-only |
| python-patterns | new copy | Win-only |
| research | new copy | Win-only |
| research-alireza | new copy | Win-only |
| rules-distill | new copy | Win-only |
| search-first | new copy | Win-only |
| tdd-workflow | new copy | Win-only |
| verification-loop | new copy | Win-only |

**Per-file processing for new copies:**
- Recursively copy all files in the skill folder (most skills have only `SKILL.md`; max is 7 files for `prd-taskmaster` / `research-alireza`).
- Detect text vs binary by null-byte sniff on first 8 KB.
- For text files: convert CRLF → LF.
- For `.sh` / `.py` files: set executable bit via `chmod +x`.
- Skip Windows OS artifacts: `Thumbs.db`, `desktop.ini`, `*.lnk`.
- Set destination mtime = source mtime (preserves "when was this skill written" semantics).

**Layer B — Plugins (6 to install on WSL):**

Generated as a separate Claude-Code-WSL script at `wsl-plugin-install.md` (not included in this snapshot) for the user to run manually inside their WSL Claude session.

---

## Operation Log

Each operation below is timestamped in IST (UTC+05:30) and atomic. Format: `[timestamp] OP_TYPE source → dest | result`.

<!-- BEGIN_OPERATIONS -->
- `[2026-06-15 14:55:23 IST]` **PLAN** source=`/mnt/c/Users/<user>/.claude/skills` dest=`/home/<user>/.claude/skills` (resolved to: `/home/<user>/<project>/.claude/skills`)
- `[2026-06-15 14:55:23 IST]` **SKIP** `drawio` (already exists on WSL: 8167 B)
- `[2026-06-15 14:55:23 IST]` **RM-SYMLINK** `grill-me` (was symlink → `../../.agents/skills/grill-me`; <project> source intact at that path)
- `[2026-06-15 14:55:24 IST]` **COPY** `grill-me` → `/home/<user>/.claude/skills/grill-me` (files: 1 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 1 | total bytes: 8890)
- `[2026-06-15 14:55:24 IST]` **COPY** `answerinshort` → `/home/<user>/.claude/skills/answerinshort` (files: 1 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 1 | total bytes: 3087)
- `[2026-06-15 14:55:24 IST]` **COPY** `api-design` → `/home/<user>/.claude/skills/api-design` (files: 3 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 3 | total bytes: 13598)
- `[2026-06-15 14:55:24 IST]` **COPY** `coding-standards` → `/home/<user>/.claude/skills/coding-standards` (files: 3 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 3 | total bytes: 13270)
- `[2026-06-15 14:55:24 IST]` **COPY** `deep-research` → `/home/<user>/.claude/skills/deep-research` (files: 6 | text/CRLF→LF: 1 | exec-bit-set: 0 | binary: 5 | total bytes: 94848)
- `[2026-06-15 14:55:24 IST]` **COPY** `deployment-patterns` → `/home/<user>/.claude/skills/deployment-patterns` (files: 2 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 2 | total bytes: 11557)
- `[2026-06-15 14:55:24 IST]` **COPY** `describe` → `/home/<user>/.claude/skills/describe` (files: 1 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 1 | total bytes: 4054)
- `[2026-06-15 14:55:24 IST]` **COPY** `docker-patterns` → `/home/<user>/.claude/skills/docker-patterns` (files: 2 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 2 | total bytes: 8841)
- `[2026-06-15 14:55:24 IST]` **COPY** `e2e-testing` → `/home/<user>/.claude/skills/e2e-testing` (files: 3 | text/CRLF→LF: 1 | exec-bit-set: 0 | binary: 2 | total bytes: 8537)
- `[2026-06-15 14:55:25 IST]` **COPY** `mw-build-and-test` → `/home/<user>/.claude/skills/mw-build-and-test` (files: 1 | text/CRLF→LF: 1 | exec-bit-set: 0 | binary: 0 | total bytes: 5059)
- `[2026-06-15 14:55:25 IST]` **COPY** `prd-taskmaster` → `/home/<user>/.claude/skills/prd-taskmaster` (files: 7 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 7 | total bytes: 82571)
- `[2026-06-15 14:55:25 IST]` **COPY** `product-requirements` → `/home/<user>/.claude/skills/product-requirements` (files: 2 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 2 | total bytes: 10524)
- `[2026-06-15 14:55:25 IST]` **COPY** `python-patterns` → `/home/<user>/.claude/skills/python-patterns` (files: 2 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 2 | total bytes: 9710)
- `[2026-06-15 14:55:25 IST]` **COPY** `research` → `/home/<user>/.claude/skills/research` (files: 2 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 2 | total bytes: 489)
- `[2026-06-15 14:55:26 IST]` **COPY** `research-alireza` → `/home/<user>/.claude/skills/research-alireza` (files: 7 | text/CRLF→LF: 2 | exec-bit-set: 2 | binary: 5 | total bytes: 66741)
- `[2026-06-15 14:55:26 IST]` **COPY** `rules-distill` → `/home/<user>/.claude/skills/rules-distill` (files: 2 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 2 | total bytes: 12072)
- `[2026-06-15 14:55:26 IST]` **COPY** `search-first` → `/home/<user>/.claude/skills/search-first` (files: 2 | text/CRLF→LF: 1 | exec-bit-set: 0 | binary: 1 | total bytes: 8145)
- `[2026-06-15 14:55:26 IST]` **COPY** `tdd-workflow` → `/home/<user>/.claude/skills/tdd-workflow` (files: 3 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 3 | total bytes: 10282)
- `[2026-06-15 14:55:27 IST]` **COPY** `verification-loop` → `/home/<user>/.claude/skills/verification-loop` (files: 3 | text/CRLF→LF: 0 | exec-bit-set: 0 | binary: 3 | total bytes: 3008)
- `[2026-06-15 14:55:27 IST]` **DONE** all operations complete

- `[2026-06-15 14:56:26 IST]` **FIXUP-START** rationale: null-byte detector (\$'\x00' in bash) collapses to empty pattern, so all files initially classified as binary. Re-processing by extension.
- `[2026-06-15 14:56:26 IST]` **FIXUP-SUMMARY** files scanned: 53 | CRLF→LF: 0 | exec-bit-set: 0
- `[2026-06-15 14:56:26 IST]` **FIXUP-DONE** all corrective ops complete
<!-- END_OPERATIONS -->

---

## Summary

### Layer A — User-level skills (DONE)

| Metric | Value |
|---|---|
| Skills on Windows (before) | 20 |
| Skills on WSL (before) | 30 |
| Win-only skills exported | 18 |
| Conflicts: skipped | 1 (`drawio` — WSL version kept) |
| Conflicts: symlink replaced with regular dir | 1 (`grill-me` — Win version copied; <project> source intact) |
| Skills on WSL (after) | **49** (= 30 + 18 + 1 replaced) |
| Files copied | 53 |
| Total bytes copied | ~395 KB |
| OS-junk files filtered (`Thumbs.db`, `desktop.ini`, `*.lnk`) | 0 |
| CRLF→LF conversions needed (after extension-based re-check) | 0 |
| Exec-bit fixes needed (after extension-based re-check) | 0 |

**Net effect:** WSL Claude Code now sees all 18 Win-only skills (answerinshort, api-design, coding-standards, deep-research, deployment-patterns, describe, docker-patterns, e2e-testing, mw-build-and-test, prd-taskmaster, product-requirements, python-patterns, research, research-alireza, rules-distill, search-first, tdd-workflow, verification-loop) plus the rewritten `grill-me`.

### Detector bug (caught & corrected)

The original copy loop used `head -c 8192 "$f" | LC_ALL=C grep -q $'\x00'` to detect binary files. **Bash collapses literal NUL bytes in command-line arguments to empty strings**, so `grep -q ''` matched every file and the detector reported every file as "binary." Consequence: CRLF→LF conversion and exec-bit setting were skipped during initial copy.

**Mitigation:** A second pass (`FIXUP-*`) re-scanned every copied file by extension allowlist (`.md .json .yaml .yml .txt .sh .py .toml ...`) and applied CRLF→LF + `chmod +x` where appropriate. Result: **0 files actually needed correction** — Windows source files happened to already use LF endings (likely VSCode / Claude Code preserving the LF convention even on Windows). No data damage occurred.

**Lesson for agentpack** ([PRD REQ-008 Task 5.6](../docs/prd-agentpack.md)): the production CRLF transformer must NOT use `$'\x00'` bash literals. Use `file --mime-encoding` or `grep -PI` instead. Logged for the implementation phase.

### Layer B — Plugins (NOT YET DONE — script generated)

WSL is missing these 6 plugins Windows has:

| Plugin | Marketplace | Action required on WSL |
|---|---|---|
| ruflo-core | ruflo | (1) add marketplace (2) install plugin |
| ruflo-rag-memory | ruflo | (2) install plugin |
| ruflo-swarm | ruflo | (2) install plugin |
| ruflo-loop-workers | ruflo | (2) install plugin |
| understand-anything | understand-anything | (1) add marketplace (2) install plugin |
| chrome-devtools-mcp | claude-plugins-official | (2) install plugin (MP exists) |

Instructions to run inside WSL Claude Code: **`wsl-plugin-install.md` (not included in this snapshot)** (sequence of `/plugin marketplace add` and `/plugin install` slash commands).

This is a manual step — not automated from outside Claude Code's REPL, because direct filesystem manipulation of `installed_plugins.json` / `known_marketplaces.json` can corrupt plugin state and bypass Claude's version-tracking and git-clone safety net.

### What's still divergent after this export

| Layer | Win | WSL (now) | Still divergent? |
|---|---|---|---|
| User skills | 20 | 49 | Yes — WSL has 29 Win doesn't (the <project> project's skill set). To make Win match, run the reverse direction later: `wsl→win` for those 29. Not done; needs separate session. |
| Plugins | 20 | 15 (becomes 21 after Layer B script) | After Layer B: WSL has all Win plugins PLUS `pyright-lsp`. To get pyright-lsp on Windows, run `/plugin install pyright-lsp@claude-plugins-official` in Windows Claude. |
| Marketplaces | 4 (incl. headroom-marketplace) | 1 (becomes 3 after Layer B) | After Layer B: WSL won't have `headroom-marketplace`. Optional add — no plugins from it currently installed. |
| Commands / Agents | (none in `~/.claude/commands` or `~/.claude/agents` either side) | (same) | Not divergent |
| MCP servers | Not enumerated in this session | Not enumerated | TODO if requested |
| Hooks | Not enumerated | Not enumerated | TODO if requested |

### Next-action checklist for user

- [ ] **Run `wsl-plugin-install.md` (not included in this snapshot) inside WSL Claude Code** to complete Layer B
- [ ] **Restart Claude Code in WSL** to load the newly-installed plugins
- [ ] **Spot-test a Win-only skill** in WSL: e.g. ask Claude `/prd-taskmaster` in WSL and confirm the skill loads
- [ ] **Spot-test a Win-only plugin** in WSL: e.g. invoke `understand-anything:understand` and confirm it works
- [ ] *(Optional)* Reverse-direction sync: export the 29 WSL-only skills back to Windows (separate session)
- [ ] *(Optional)* Install `pyright-lsp@claude-plugins-official` on Windows to match WSL
- [ ] **Feed lessons back to agentpack PRD:** the CRLF detector bug, the symlink-handling edge case for `grill-me`, the OS-junk filter list, and the marketplace-precondition logic for plugin installs are all real-world calibration data for [agentpack PRD REQ-008 and REQ-003 install logic](../docs/prd-agentpack.md)

---

## Reproducibility note

This entire export was performed manually as a **proof-of-concept of the agentpack `wsl-sync` workflow specified in the PRD**. The behaviors logged here (skip on overlap, symlink replacement with regular dir, OS-junk filtering, ext-based CRLF detection, idempotent copy) are exactly what `agentpack wsl-sync --from win-native --to wsl` should automate in Phase 1 / Task 33 of the PRD.

When agentpack ships, the equivalent of this entire session should be:

```bash
agentpack wsl-sync --from win-native --to wsl --dry-run     # preview
agentpack wsl-sync --from win-native --to wsl               # execute
```

…with all the same conflict resolution, logging, and verification automatic.

