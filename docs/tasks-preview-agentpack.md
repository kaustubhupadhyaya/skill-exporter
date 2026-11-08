# Tasks Preview — agentpack Phase 1 MVP

Hand-prepared task breakdown derived from [prd-agentpack.md](./prd-agentpack.md). Matches the structure TaskMaster would produce on `parse-prd --num-tasks 47` so it can be imported manually into any tracker (GitHub Issues / Linear / Jira) until TaskMaster is installed.

**Phase 1 totals:** 47 implementation tasks + 7 USER-TEST checkpoints = **54 tasks**, ~184 unbuffered hours, ~230 buffered hours (~10–12 weeks part-time at ~20h/week).

---

## Legend

- **Complexity:** Trivial (≤1h) / Small (2–4h) / Medium (5–8h) / Large (9–16h)
- **REQ:** Functional requirement number it implements
- **Deps:** Task IDs that must finish first
- **Priority:** P0 (blocks MVP) / P1 (important) / P2 (polish)

---

## Week 1–2 — Foundation

### Task 1: Initialize Go module + linting + CI matrix
- **Complexity:** Small (3h) — **P0**
- **REQ:** Infrastructure
- **Deps:** None
- **Acceptance:**
  - [ ] `go.mod` with module path `github.com/<user>/agentpack`, Go 1.22+
  - [ ] `golangci-lint` config committed (`.golangci.yml`)
  - [ ] GitHub Actions workflow: matrix `{ubuntu, macos, windows} × {build, test, lint}` passes on empty stub
  - [ ] CONTRIBUTING.md + LICENSE (Apache-2.0 recommended)

### Task 2: Define `manifest.v1.json` JSON Schema + YAML loader + validator
- **Complexity:** Medium (5h) — **P0**
- **REQ:** REQ-001
- **Deps:** Task 1
- **Acceptance:**
  - [ ] `schemas/manifest.v1.json` validates a hand-written valid manifest
  - [ ] Schema rejects: bad version, malformed name (non-kebab), missing required fields
  - [ ] Go struct `internal/bundle/Manifest` round-trips YAML→struct→YAML losslessly
  - [ ] Unit tests cover at least 5 valid and 5 invalid fixtures

### Task 3: Implement reproducible tar+zstd bundle writer
- **Complexity:** Medium (5h) — **P0**
- **REQ:** REQ-001
- **Deps:** Task 2
- **Acceptance:**
  - [ ] `internal/bundle.Write(w, manifest, artifacts)` produces tar+zstd
  - [ ] Sorted entry order, mtime=0, mode normalized to 0644/0755 by `is_executable` flag
  - [ ] Property test: same inputs → same SHA-256 across 100 runs
  - [ ] Bundle survives unzst+untar round-trip

### Task 4: Implement bundle reader + `agentpack inspect`
- **Complexity:** Small (4h) — **P0**
- **REQ:** REQ-001
- **Deps:** Task 3
- **Acceptance:**
  - [ ] `internal/bundle.Read(r)` returns manifest + artifact iterator
  - [ ] `agentpack inspect <bundle>` prints manifest + artifact tree + total bytes
  - [ ] Path-traversal protection: refuses any entry with `..` or absolute path

### Task 5: Set up `urfave/cli/v3` skeleton with subcommand stubs
- **Complexity:** Small (2h) — **P0**
- **REQ:** Infrastructure
- **Deps:** Task 1
- **Acceptance:**
  - [ ] `agentpack --help` lists: export, install, inspect, lint, verify, sign, keygen, secrets, wsl-sync, diff
  - [ ] Each stub returns "not implemented" with non-zero exit

### **USER-TEST 1** (after Task 5)
- **Description:** Manually craft a small bundle, run `agentpack inspect`, verify output is correct and SHA-256 is reproducible on rebuild.
- **Estimated time:** 30 min

---

## Week 3–4 — Export

### Task 6: Implement Claude Code scanner
- **Complexity:** Medium (6h) — **P0**
- **REQ:** REQ-002
- **Deps:** Task 4
- **Acceptance:**
  - [ ] `internal/scanner/claude.Scan(dir)` returns: skills, slash_commands, subagents, MCP entries
  - [ ] Reads from: `~/.claude/skills/`, `~/.claude/commands/`, `~/.claude/agents/`, `~/.claude.json`, `~/.claude/settings.json` (MCP key only in MVP), project `.mcp.json`
  - [ ] Per-OS path resolution works on Win, Mac, Linux
  - [ ] Returns artifact paths relative to scan root

### Task 7: Implement inlined-secret detector
- **Complexity:** Medium (5h) — **P0**
- **REQ:** REQ-007
- **Deps:** None (independent)
- **Acceptance:**
  - [ ] Detects: AWS access keys (`AKIA…`), GitHub PATs (`ghp_…`), generic high-entropy strings (≥40 chars, Shannon ≥4.5)
  - [ ] Returns location (artifact path + field/line) on hit
  - [ ] Allowlist file support `~/.config/agentpack/secrets-allowlist.txt`
  - [ ] Test fixture with known-bad MCP config → detector returns hit

### Task 8: Wire `agentpack export <dir>` end-to-end
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-002
- **Deps:** Tasks 6, 7
- **Acceptance:**
  - [ ] `agentpack export ~/.claude --out my.tar.zst` produces valid bundle
  - [ ] If secret detected, exit 2 with helpful message
  - [ ] If `--allow-inlined-secrets`, prints big warning and proceeds

### Task 9: Implement `--dry-run` planner with tree printer
- **Complexity:** Small (2h) — **P0**
- **REQ:** REQ-002
- **Deps:** Task 8
- **Acceptance:**
  - [ ] `agentpack export ~/.claude --dry-run` prints artifact tree with byte sizes; writes nothing
  - [ ] Per-artifact-type subtotals printed

### Task 10: Integration test — real fixture → export → inspect
- **Complexity:** Small (4h) — **P0**
- **REQ:** REQ-002
- **Deps:** Task 9
- **Acceptance:**
  - [ ] `tests/fixtures/claude-30-artifacts/` checked into repo
  - [ ] Test exports, inspects, asserts artifact counts and key hashes match expected

### **USER-TEST 2** (after Task 10)
- **Description:** Export your real Claude Code config (`~/.claude/`) into a bundle, run inspect, confirm all skills/commands/agents you expect are present.
- **Estimated time:** 30 min

---

## Week 5–6 — Translation & Install

### Task 11: Define canonical internal MCP schema + Claude reader/writer
- **Complexity:** Small (4h) — **P0**
- **REQ:** REQ-004
- **Deps:** Task 2
- **Acceptance:**
  - [ ] `internal/mcp.Server` struct covers transport/command/args/env/url/headers/auth
  - [ ] `mcp.ReadClaudeJSON(r)` → []Server; `mcp.WriteClaudeJSON(w, []Server)` round-trips
  - [ ] Unknown JSON fields preserved through round-trip

### Task 12: Codex TOML reader/writer + env-var conversion
- **Complexity:** Medium (5h) — **P0**
- **REQ:** REQ-004
- **Deps:** Task 11
- **Acceptance:**
  - [ ] `mcp.ReadCodexTOML(r)`, `mcp.WriteCodexTOML(w, []Server)` round-trip Codex's `[mcp_servers]` shape
  - [ ] `${VAR}` ↔ literal value conversion via Codex's `${VAR:-default}` semantics
  - [ ] Test fixture: a real Codex config round-trips byte-identically

### Task 13: opencode JSONC reader/writer with comment preservation
- **Complexity:** Medium (5h) — **P0**
- **REQ:** REQ-004
- **Deps:** Task 11
- **Acceptance:**
  - [ ] Use `tidwall/jsonc` or equivalent; comments preserved on read+write
  - [ ] Writes `mcp` key (not `mcpServers`), `type: local/remote`, `command: [...]` array form, `environment` not `env`
  - [ ] Test fixture: opencode official config snippet round-trips

### Task 14: Cursor JSON reader/writer
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-004
- **Deps:** Task 11
- **Acceptance:**
  - [ ] Writes `~/.cursor/mcp.json` shape
  - [ ] Env-var syntax: `${VAR}` like Claude
  - [ ] Round-trip test

### Task 15: Claude installer
- **Complexity:** Medium (6h) — **P0**
- **REQ:** REQ-003
- **Deps:** Task 11
- **Acceptance:**
  - [ ] Writes skills/commands/agents under `<into>/skills/<n>/`, `<into>/commands/<n>.md`, `<into>/agents/<n>.md`
  - [ ] Writes MCP entries to `<into>/.claude.json` merging with existing
  - [ ] Atomic writes (`.tmp` + rename)

### Task 16: Codex installer
- **Complexity:** Small (4h) — **P0**
- **REQ:** REQ-003
- **Deps:** Task 12
- **Acceptance:**
  - [ ] Writes only MCP entries (Codex has no skills concept) into `~/.codex/config.toml`
  - [ ] Atomic write

### Task 17: opencode + Cursor installers
- **Complexity:** Medium (6h) — **P0**
- **REQ:** REQ-003
- **Deps:** Tasks 13, 14
- **Acceptance:**
  - [ ] opencode installer writes skills (where supported) + MCP entries
  - [ ] Cursor installer writes MCP entries only
  - [ ] Atomic writes for both

### Task 18: Collision policy implementation
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-003
- **Deps:** Task 15
- **Acceptance:**
  - [ ] `--on-collision backup|skip|overwrite|fail` flag
  - [ ] Backup: rename `<file>` to `<file>.agentpack-backup-<RFC3339-ts>`
  - [ ] Default is `backup`

### Task 19: Wire `agentpack install` end-to-end
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-003
- **Deps:** Tasks 15–18
- **Acceptance:**
  - [ ] `agentpack install <bundle> --target <agent>` dispatches to per-target installer
  - [ ] Prints summary after install (N skills, M MCPs, etc.)
  - [ ] `--dry-run` shows plan without writing

### **USER-TEST 3** (after Task 19)
- **Description:** Round-trip on your real setup: export Claude → install into Codex (or Cursor) → manually verify the MCP entries appear in the right file with the right syntax.
- **Estimated time:** 30 min

---

## Week 7 — Secrets

### Task 20: `SecretBackend` trait + URI parser
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-005
- **Deps:** Task 2
- **Acceptance:**
  - [ ] `internal/secrets.URI` struct: backend, path, query
  - [ ] Parser handles `secret://env/VAR` and `secret://keychain/service/account` correctly
  - [ ] Trait `SecretBackend { Name(), Resolve(uri), Has(uri) }` defined
  - [ ] Malformed URIs return clear error

### Task 21: env backend
- **Complexity:** Trivial (1h) — **P0**
- **REQ:** REQ-005
- **Deps:** Task 20
- **Acceptance:**
  - [ ] Reads from `os.Getenv`; returns Unresolved error if unset
  - [ ] Round-trip test

### Task 22: Windows DPAPI / Credential Manager backend
- **Complexity:** Medium (5h) — **P0**
- **REQ:** REQ-005
- **Deps:** Task 20
- **Acceptance:**
  - [ ] Uses `danieljoos/wincred` or equivalent
  - [ ] Build-tag `windows` only
  - [ ] Round-trip: write, read, delete works on Win runner

### Task 23: macOS Keychain backend
- **Complexity:** Medium (5h) — **P0**
- **REQ:** REQ-005
- **Deps:** Task 20
- **Acceptance:**
  - [ ] Uses `keybase/go-keychain` or equivalent
  - [ ] Build-tag `darwin` only
  - [ ] Round-trip test on Mac runner

### Task 24: Linux libsecret backend
- **Complexity:** Medium (5h) — **P0**
- **REQ:** REQ-005
- **Deps:** Task 20
- **Acceptance:**
  - [ ] Uses `zalando/go-keyring` or Secret Service D-Bus directly
  - [ ] Build-tag `linux` only
  - [ ] CI runs gnome-keyring in headless mode for round-trip test

### Task 25: `agentpack secrets list / bind` CLI
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-005
- **Deps:** Task 20
- **Acceptance:**
  - [ ] `agentpack secrets list <bundle>` shows all `secret://` refs with resolved/unresolved status
  - [ ] `agentpack secrets bind <ref> <new-uri>` updates `~/.config/agentpack/bindings/<bundle>.yaml`

### Task 26: Wire secret resolution into install pipeline
- **Complexity:** Small (4h) — **P0**
- **REQ:** REQ-005
- **Deps:** Tasks 21–25, Task 19
- **Acceptance:**
  - [ ] Install detects unresolved secrets, exits 4 with list
  - [ ] Default mode: emit env-var reference in target config (lazy resolution)
  - [ ] Test: `secret://env/FOO` in bundle → `${FOO}` in installed Claude/Cursor MCP, `${FOO}` in Codex TOML

### **USER-TEST 4** (after Task 26)
- **Description:** Set an env var, install a bundle that references it, verify resolution. Repeat with a keychain entry on your actual OS keychain.
- **Estimated time:** 45 min

---

## Week 8 — Signing + WSL bridge

### Task 27: `agentpack keygen`
- **Complexity:** Small (2h) — **P0**
- **REQ:** REQ-006
- **Deps:** Task 3
- **Acceptance:**
  - [ ] Generates ed25519 keypair, minisign format
  - [ ] Writes private key with 0600 mode (warns on Windows where permissions are different)
  - [ ] Prints public key to stdout

### Task 28: `agentpack sign / verify`
- **Complexity:** Small (4h) — **P0**
- **REQ:** REQ-006
- **Deps:** Task 27
- **Acceptance:**
  - [ ] Uses `jedisct1/go-minisign` or equivalent
  - [ ] `sign` produces `.minisig` sidecar OR embedded `signatures/manifest.minisig` in bundle
  - [ ] `verify --public-key X` returns 0 on valid, exit 3 on tamper

### Task 29: Install-time signature verification
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-006
- **Deps:** Tasks 28, 19
- **Acceptance:**
  - [ ] If bundle has signature and no `--public-key` provided → refuse
  - [ ] `--public-key-url` only over HTTPS; fetch + verify
  - [ ] Tampering test: modify bundle bytes, install fails with exit 3

### Task 30: WSL detection on Windows host
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-008
- **Deps:** Task 1
- **Acceptance:**
  - [ ] `internal/wsl.ListDistros()` parses `wsl.exe -l -v` output
  - [ ] Returns name + state (Running/Stopped) + WSL version
  - [ ] Errors clearly when running on non-Windows host

### Task 31: Win ↔ WSL path translation
- **Complexity:** Small (2h) — **P0**
- **REQ:** REQ-008
- **Deps:** Task 30
- **Acceptance:**
  - [ ] `WinToWsl(distro, "C:\\Users\\x\\.claude")` → `"/mnt/c/Users/x/.claude"` (or detects `\\wsl$\…` for distro home)
  - [ ] `WslToWin(distro, "/home/x/.claude")` → `"\\wsl$\<distro>\home\x\.claude"`

### Task 32: CRLF↔LF transformer with binary detection
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-008
- **Deps:** None
- **Acceptance:**
  - [ ] Only transforms files with extensions in allowlist (`.md`, `.sh`, `.py`, `.js`, `.ts`, `.txt`, `.json`, `.yaml`, `.yml`)
  - [ ] Refuses to transform if null byte detected in first 8KB
  - [ ] Property test: random binary content unchanged through transformer

### Task 33: Wire `agentpack wsl-sync` command
- **Complexity:** Medium (5h) — **P0**
- **REQ:** REQ-008
- **Deps:** Tasks 30–32
- **Acceptance:**
  - [ ] `agentpack wsl-sync --from win-native --to wsl [--distro X] [--dry-run]` works on Windows runner
  - [ ] Distro auto-detect if `--distro` omitted (picks first running WSL2 distro)
  - [ ] Output summary: N files copied, M transformed, K skipped (already up-to-date)
  - [ ] Idempotent: 2nd run with no changes = 0 writes

### Task 34: OneDrive collision detection
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-008
- **Deps:** Task 33
- **Acceptance:**
  - [ ] Detects if source path is under OneDrive root (check known paths + `OneDrive` env var)
  - [ ] Warns with explicit message + suggests exclusion or move
  - [ ] User can `--force` to proceed anyway

### **USER-TEST 5** (after Task 34) — *the headline personal-pain test*
- **Description:** On your actual Windows+WSL machine: install a skill on Windows-native Claude Code → run `agentpack wsl-sync --from win-native --to wsl` → open Claude Code inside WSL → verify the skill appears.
- **Estimated time:** 30 min

---

## Week 9 — Lint + remote bundle fetch + diff

### Task 35: Lint rule registry trait
- **Complexity:** Small (3h) — **P1**
- **REQ:** REQ-009
- **Deps:** Task 4
- **Acceptance:**
  - [ ] `LintRule` interface: ID(), Severity(), Description(), Check(bundle)
  - [ ] Registry registration + ordering
  - [ ] `--rules <list>` and `--ignore <id>` flags supported

### Task 36: Implement 5 MVP lint rules
- **Complexity:** Medium (6h) — **P1**
- **REQ:** REQ-009
- **Deps:** Task 35
- **Acceptance:**
  - [ ] `inlined-secret` (P0): bundle MUST NOT contain plaintext that matches secret patterns
  - [ ] `invalid-manifest-schema` (P0): manifest passes JSON Schema
  - [ ] `missing-referenced-file` (P0): every artifact in manifest exists in tar
  - [ ] `unsupported-target-artifact` (P1): warn if bundle declares Codex target but contains skills (Codex doesn't load them)
  - [ ] `non-portable-absolute-path` (P1): any path starting with `C:\` or `/home/specific-user/` warns

### Task 37: `agentpack lint` CLI with `--json` output
- **Complexity:** Small (3h) — **P1**
- **REQ:** REQ-009
- **Deps:** Task 36
- **Acceptance:**
  - [ ] `agentpack lint <bundle>` prints human-readable results
  - [ ] `--json` outputs machine-readable
  - [ ] Exit code: 0 clean, 1 warnings, 2 errors

### Task 38: `github://owner/repo@tag` URL fetcher
- **Complexity:** Small (4h) — **P0**
- **REQ:** REQ-003
- **Deps:** Task 19
- **Acceptance:**
  - [ ] Parser: `github://kaustubh/agentpack-bundle@v0.1.0` → resolve to release asset URL
  - [ ] Asset name convention: `<repo-basename>.agentpack.tar.zst`
  - [ ] Fetch via GitHub REST API, cache in `~/.cache/agentpack/`

### Task 39: HTTPS fetcher with content-hash verification
- **Complexity:** Small (3h) — **P0**
- **REQ:** REQ-003
- **Deps:** Task 38
- **Acceptance:**
  - [ ] Direct HTTPS URL accepted by `install`
  - [ ] Reject HTTP (exit 1)
  - [ ] If `--sha256 <hex>` provided, verify after download
  - [ ] Cache by URL+hash

### Task 40: `agentpack diff` command
- **Complexity:** Medium (5h) — **P1**
- **REQ:** Story 4
- **Deps:** Task 4
- **Acceptance:**
  - [ ] `agentpack diff a.tar.zst b.tar.zst` shows added/removed/modified artifacts
  - [ ] Per-artifact type counts
  - [ ] Exit 0 if identical, 1 if different

### **USER-TEST 6** (after Task 40)
- **Description:** Publish a bundle to GitHub Releases (under your test repo), install via `github://you/repo@v0.0.1`, run diff against a modified version.
- **Estimated time:** 30 min

---

## Week 10 — Release prep

### Task 41: Cross-compile release pipeline
- **Complexity:** Medium (5h) — **P0**
- **REQ:** Goal 3
- **Deps:** All MVP features
- **Acceptance:**
  - [ ] GitHub Actions release workflow on tag push
  - [ ] Builds: win-x64, win-arm64, darwin-x64, darwin-arm64, linux-x64, linux-arm64
  - [ ] Uses `-trimpath -buildvcs=false` for reproducibility
  - [ ] Uploads as GitHub Release assets

### Task 42: Sign release binaries via OIDC + minisign
- **Complexity:** Medium (5h) — **P0**
- **REQ:** REQ-006
- **Deps:** Tasks 28, 41
- **Acceptance:**
  - [ ] Each binary has a `.minisig` sidecar published alongside
  - [ ] Public key committed to repo at `/agentpack.pub`
  - [ ] Install instructions in README verify signature before running

### Task 43: TTRC benchmark
- **Complexity:** Small (4h) — **P0**
- **REQ:** Goal 1
- **Deps:** All install/export
- **Acceptance:**
  - [ ] `benchmarks/ttrc.sh` runs export + install round-trip on fixture
  - [ ] Measures wall-clock per OS
  - [ ] Results table in `benchmarks/ttrc.md`
  - [ ] Asserts Goal 1 target (<5 min) met or documents shortfall

### Task 44: README.md
- **Complexity:** Medium (6h) — **P0**
- **REQ:** Documentation
- **Deps:** All MVP features
- **Acceptance:**
  - [ ] Install: brew / winget / curl one-liner
  - [ ] Quickstart: 3 commands max
  - [ ] Comparison table vs vsync / skillshare / sync-agents-settings
  - [ ] Link to bundle-format and secrets docs

### Task 45: `docs/bundle-format.md` spec
- **Complexity:** Small (4h) — **P0**
- **REQ:** Documentation
- **Deps:** Task 2
- **Acceptance:**
  - [ ] Tar layout
  - [ ] Manifest schema with all fields
  - [ ] Reproducibility rules
  - [ ] Versioning policy

### Task 46: `docs/secrets.md` — URI scheme + per-OS backend setup
- **Complexity:** Small (4h) — **P0**
- **REQ:** Documentation
- **Deps:** Task 26
- **Acceptance:**
  - [ ] URI grammar
  - [ ] Per-OS keychain bootstrap (Win Credential Manager, macOS Keychain, Linux gnome-keyring/KWallet)
  - [ ] Migration patterns

### Task 47: Cut 0.1.0 release + submit to Claude Code Plugin marketplace
- **Complexity:** Small (3h) — **P0**
- **REQ:** Goal 5
- **Deps:** Tasks 41–46
- **Acceptance:**
  - [ ] Git tag `v0.1.0` pushed
  - [ ] Release notes written (using "What's New" + "Known Limitations" sections)
  - [ ] `.claude-plugin/plugin.json` published, marketplace.json entry submitted
  - [ ] Announcement: launch tweet, r/ClaudeAI post, HN show-hn

### **USER-TEST 7** (after Task 47)
- **Description:** Spin up a fresh VM (cloud or local). Follow README quickstart exactly. Time from `nothing` to `installed bundle from github://`.
- **Estimated time:** 30 min

---

## Task summary

- **Implementation tasks:** 47
- **USER-TEST checkpoints:** 7
- **Total:** 54
- **Phase 1 raw effort:** ~184 hours
- **With 25% buffer:** ~230 hours
- **At ~20h/week:** ~11–12 weeks

## Parallelizable groups

The following tasks can run in parallel once predecessors are done:

- **Group A** (Translators, after Task 11): 12, 13, 14
- **Group B** (Per-OS keychain backends, after Task 20): 22, 23, 24
- **Group C** (Docs vs cross-compile, after install features): 41 vs 44 vs 45 vs 46
- **Group D** (Lint rules, after Task 35): 36 has 5 sub-rules that can be split

## Critical path

1 → 2 → 3 → 6 → 8 → 11 → 15 → 19 → 20 → 26 → 33 → 41 → 47

(~70 hours of critical-path work; the rest is fan-out parallelizable)

---

## Importing into TaskMaster (once installed)

```bash
# 1. Install
npm install -g task-master-ai

# 2. Initialize in this repo (if not already)
task-master init --yes

# 3. Parse the PRD
task-master parse-prd --input .taskmaster/docs/prd.md --research --num-tasks 47

# 4. Expand into subtasks
task-master expand-all --research

# 5. Add USER-TEST checkpoints (manual)
task-master add-task --title "USER-TEST 1: Inspect a hand-crafted bundle" --priority high --dependencies 5
# ... etc
```

## Importing into GitHub Issues (alternative)

Each task above can be converted to a GitHub Issue with:
- Title: `[Task N] <task title>`
- Labels: `phase-1`, `mvp`, `priority:P0/P1`, complexity tag
- Body: the acceptance criteria as a checklist
- Use the `gh` CLI to bulk-create from a script if desired

---

End of tasks preview.
