# PRD: agentpack — Portable AI Agent Configuration Bundles

**Author:** kaustubhupadhyaya
**Date:** 2026-06-14
**Status:** Draft
**Version:** 1.0
**Taskmaster Optimized:** Yes
**Source Research:** [research/skill-exporter-research-brief.md](../research/skill-exporter-research-brief.md)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Problem Statement](#problem-statement)
3. [Goals & Success Metrics](#goals--success-metrics)
4. [User Stories](#user-stories)
5. [Functional Requirements](#functional-requirements)
6. [Non-Functional Requirements](#non-functional-requirements)
7. [Technical Considerations](#technical-considerations)
8. [Implementation Roadmap](#implementation-roadmap)
9. [Out of Scope](#out-of-scope)
10. [Open Questions & Risks](#open-questions--risks)
11. [Validation Checkpoints](#validation-checkpoints)
12. [Appendix: Task Breakdown Hints](#appendix-task-breakdown-hints)

---

## Executive Summary

Developers using multiple AI coding agents (Claude Code, Codex CLI, opencode, Cursor) on multiple environments (Windows, WSL, Linux, macOS) are forced to install and configure skills, MCP servers, hooks, and slash commands separately in each surface — and existing sync tools cover only fragments of the problem while [24,008 secrets leaked from MCP configs on public GitHub in 2026](https://www.token.security/blog/how-to-stop-exposing-secrets-on-your-mcp-configs). `agentpack` is a single CLI that exports any developer's full agent configuration surface into a **signed, secrets-aware, portable bundle** that can be installed safely into any supported agent on any OS. The MVP solves four boundaries (OS, agent, device, team) with a phased build that ships value in 6–10 weeks, the full Phase 1+2 surface in ~16 weeks, and the bidirectional sync moonshot in Phase 3.

---

## Problem Statement

### Current Situation

A developer who runs Claude Code on Windows-native PowerShell, Claude Code inside WSL, OpenAI Codex CLI, opencode, and Cursor — even just on a single laptop — currently has to install every skill, every MCP server, every hook, and every slash command **separately in each of those surfaces**. The configs live in different directories, in different formats (JSON / JSONC / TOML), with different env-var syntaxes (`${VAR}` / `${env:VAR}` / `{env:VAR}` / `$VAR`), and there is no automatic mechanism to keep them in sync. The same problem multiplies across devices (laptop ↔ desktop ↔ remote dev box) and across team members.

Existing sync tools (4 reviewed: [skillshare](https://github.com/runkids/skillshare) 2.2k★, [vsync](https://github.com/nicepkg/vsync), [sync-agents-settings](https://github.com/Leoyang183/sync-agents-settings), [antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills) 40k★) each cover a subset — none of them cover MCPs + skills + hooks + slash commands + subagents simultaneously, **none of them bridge the Windows ↔ WSL filesystem split**, and **none of them treat secrets as a first-class concern**.

### User Impact

- **Who is affected:**
  - Solo developers running multiple AI CLIs on the same machine (the modal Claude Code + Cursor + Codex user)
  - Developers on Windows + WSL combo (~5M+ WSL monthly active users; subset using AI CLIs)
  - Team leads who want to publish a curated skill bundle for a team of 3–20
  - OSS skill authors who want their work usable across all agents without writing 5 install guides

- **How they're affected:**
  - **3× config maintenance overhead**: install a new MCP server → must repeat in Claude, Cursor, Codex separately
  - **Silent drift**: a hook works in Claude on Windows but breaks in Claude on WSL because of CRLF/LF or `bash` vs `pwsh`
  - **Lost work on machine switch**: a custom subagent crafted on the laptop is invisible from the desktop
  - **Onboarding friction**: a new teammate cannot import a curated config in one command
  - **Active secret leaks** (see Business Impact)

- **Severity: HIGH (for the affected segment), based on:**
  - Anthropic open feature request [Issue #49933 (WSL Remote Integration)](https://github.com/anthropics/claude-code/issues/49933) — unresolved, multiple workarounds enumerated and rejected by users
  - Documented per-environment storage (`~/.claude/` in WSL vs `C:\Users\<u>\.claude\` in Windows native — two separate filesystems on the same machine, no documented sync)
  - 4 independent sync tools shipping in last 12 months — market validation
  - Skillshare alone gathered 2.2k stars in <6 months

### Business Impact

- **Cost of problem (security):**
  - **24,008 unique secrets exposed in MCP configs on public GitHub in 2026** ([GitGuardian / Token Security 2026](https://www.token.security/blog/how-to-stop-exposing-secrets-on-your-mcp-configs))
  - **2,117 confirmed live valid credentials** capable of authenticating against real services
  - **AI-assisted commits leak secrets at 3.2% vs 1.5% baseline rate** ([Snyk State of Secrets 2026](https://snyk.io/articles/state-of-secrets/)) — ~2× elevation directly attributable to AI-tool config-sharing patterns

- **Cost of problem (productivity, estimated):**
  - Manual cross-agent install averages ~5–15 min per artifact (skill / MCP / hook). Power user with 30 artifacts × 4 agents = ~10–30 hours of maintenance per quarter, per developer
  - Onboarding a teammate to a curated agent setup: ~2–4 hours documented in dotfiles blogs ([Beeming dotfiles](https://gordonbeeming.com/blog/2026-03-10/my-dotfiles-setup-how-i-manage-my-dev-environment); [Mowinckel blog](https://drmowinckels.io/blog/2026/dotfiles-coding-agents/))

- **Strategic importance (for the builder):**
  - User's stated public-repo goal — solves a real, validated, named pain experienced personally
  - First-mover on the secrets-aware bundle niche (no competitor handles it)
  - Aligns with converging standards (AGENTS.md / MCP Registry under Linux Foundation governance)

### Why Solve This Now?

1. **MCP Registry just shipped** (Sept 2025, Linux Foundation Dec 2025) — discovery is now a standard; local config sync is the next layer.
2. **AGENTS.md adoption proved cross-vendor coordination is possible** for instructions; the analogous gap for skills/MCPs/hooks is unsolved.
3. **Claude Code Plugin format** (`.claude-plugin/plugin.json` + `marketplace.json`) gives the first credible bundle format — agentpack can be plugin-marketplace-compatible without being plugin-only.
4. **The market is moving fast** — skillshare ships every 2 weeks; if the user waits, the gap closes. 6-week MVP timing is right.
5. **Secrets-leak numbers are public and large** — the "secrets-aware bundle" pitch has timely credibility.

---

## Goals & Success Metrics

### Goal 1: Eliminate cross-agent / cross-OS manual reinstallation

- **Description:** A developer can capture their full Claude Code (or Codex, opencode, Cursor) setup on machine/agent A and install it identically on machine/agent B with one command pair.
- **Metric:** Time-to-replicate-config (TTRC) — wall-clock minutes from "install agentpack on machine B" to "all skills/MCPs/commands/subagents from machine A available on machine B."
- **Baseline:** 60–180 minutes for ~30-artifact setup (manual, per user testimonials in dotfiles blogs)
- **Target:** **< 5 minutes** for the same setup with agentpack (90%+ reduction)
- **Timeframe:** Demonstrable at MVP launch (Week 10)
- **Measurement Method:** Documented benchmark in `benchmarks/ttrc.md`; reproducible test fixture (30 skills, 8 MCPs, 5 hooks, 3 subagents); CI-run on Win + WSL + macOS runners

### Goal 2: Zero secrets in shared bundles by construction

- **Description:** No agentpack bundle (whether `export`ed or distributed) can contain inlined secret values — references only.
- **Metric:** Secret-leak rate in bundles published using agentpack
- **Baseline:** Inferred 2–3% per published config based on [GitGuardian 2026 ratios](https://snyk.io/articles/state-of-secrets/) (24k leaks / ~1M public MCP configs sampled)
- **Target:** **0 inlined secrets** detected in bundles produced by `agentpack export` (enforced by `agentpack lint` running by default, blocking export on detection)
- **Timeframe:** Built into MVP (Week 10), with secret-scanner E2E tests in CI
- **Measurement Method:** Built-in `agentpack lint` runs gitleaks-style regex + entropy checks before bundling; bundle format spec forbids string values in `secret://` typed fields; CI fixture attempts to bundle a known-bad config and asserts failure

### Goal 3: Single-binary, fast install across all three target OSes

- **Description:** Users on Windows, macOS, and Linux can install agentpack in one command with no language runtime dependency.
- **Metric:** Install command count + install time
- **Baseline:** N/A (new tool)
- **Target:** **1 command, < 30 seconds**, no Node/Python/Go runtime required at install time (single static binary per platform; `brew install agentpack` / `winget install agentpack` / `curl ... | sh`)
- **Timeframe:** Achieved at MVP launch (Week 10)
- **Measurement Method:** GitHub Actions release pipeline produces signed binaries for win-x64, win-arm64, darwin-x64, darwin-arm64, linux-x64, linux-arm64; install instructions in README; install-time benchmarked in release notes

### Goal 4: Cross-agent fidelity (lossless round-trip for supported artifacts)

- **Description:** Exporting a Claude Code config to a bundle and installing into another Claude Code instance produces byte-identical artifacts. Cross-agent installs preserve semantics for supported fields.
- **Metric:** Round-trip fidelity test pass rate
- **Baseline:** N/A
- **Target:** **100% byte-identical** for same-agent round-trip on supported artifacts (skills, slash commands, subagents, MCP servers); **≥ 95% semantic fidelity** for cross-agent installs (MCP server entries, skill content) — measured by automated fixture suite
- **Timeframe:** MVP launch (Week 10); fidelity tests grow with each new artifact type added
- **Measurement Method:** Fixture-based round-trip test suite in `tests/fidelity/`; CI gate on regressions

### Goal 5: Community adoption (vanity but tracked)

- **Description:** Build evidence that the tool is solving a real problem at sufficient scale to validate Phase 2/3 investment.
- **Metric:** GitHub stars + monthly active installers + contributed bundle count
- **Baseline:** 0
- **Target:**
  - **100 GitHub stars within 30 days of launch** (low bar to confirm interest)
  - **500 stars within 90 days** (confirms growth)
  - **At least 5 third-party-authored agentpack bundles** published on GitHub within 90 days
- **Timeframe:** 30 / 90 days post-Week-10 launch
- **Measurement Method:** GitHub API for stars; GitHub releases download count; manual GitHub search for `topic:agentpack` repos

---

## User Stories

### Story 1: Solo dev mirrors Claude Code config from Windows-native to WSL

**As a** Windows developer running Claude Code in both PowerShell-native and WSL Ubuntu on the same laptop,
**I want to** export my entire Claude Code config from one and install it identically in the other,
**So that I can** stop manually re-installing skills/MCPs/commands every time I add something on one side.

**Acceptance Criteria:**
- [ ] `agentpack export ~/.claude --out my-claude.agentpack.tar.zst` produces a single bundle file containing all SKILL.md skills, slash commands under `~/.claude/commands/`, subagents under `~/.claude/agents/`, MCP server entries from `~/.claude.json` and `.mcp.json`, and a manifest
- [ ] Bundle is < 10 MB for a typical 30-artifact setup (compression ratio target)
- [ ] Running `agentpack install my-claude.agentpack.tar.zst --target claude` in WSL after the export installs every artifact into `~/.claude/` inside the WSL distro
- [ ] CRLF line endings in skills are auto-converted to LF on WSL install
- [ ] Hidden `.git` folders and any `.DS_Store` / `Thumbs.db` are stripped from bundles
- [ ] If a target artifact already exists with different content, default behavior is to back up the existing version to `<file>.agentpack-backup-<timestamp>` and report what was overwritten
- [ ] User can pass `--dry-run` to preview install actions without writing anything

**Task Breakdown Hint:**
- Task 1.1: Implement Claude Code artifact scanner (read `~/.claude/`) (~6h)
- Task 1.2: Implement bundle serializer (tar+zstd) with deterministic ordering (~4h)
- Task 1.3: Implement Claude Code installer with backup-on-overwrite (~6h)
- Task 1.4: Implement CRLF↔LF normalization layer (~3h)
- Task 1.5: Implement `--dry-run` flag plumbing (~2h)
- Task 1.6: Round-trip fidelity test fixture (30 artifacts, Win→WSL) (~5h)

**Dependencies:** REQ-001 (bundle format), REQ-002 (manifest schema), REQ-005 (Claude Code adapter)

---

### Story 2: Cross-agent install — share Claude MCP setup with Cursor user

**As a** developer who configures my MCP servers carefully in Claude Code,
**I want to** install the same MCP server set in Cursor on the same machine,
**So that I can** use the same external tools in both agents without re-configuring each in turn.

**Acceptance Criteria:**
- [ ] `agentpack export ~/.claude --out claude.agentpack.tar.zst --include mcp` produces a bundle containing only MCP server entries (other artifact types excluded)
- [ ] `agentpack install claude.agentpack.tar.zst --target cursor` writes MCP server entries to `~/.cursor/mcp.json` using Cursor's JSON shape (`mcpServers` key, `${VAR}` env-var syntax)
- [ ] If the source bundle used `${env:VAR}` (Cursor/opencode style) and the target is Codex, the env-var syntax is translated to `${VAR}` and TOML format
- [ ] Pre-existing MCP entries in the target file are merged-not-replaced by default; collision policy is configurable (`--on-collision skip|overwrite|fail`)
- [ ] Bundle never contains plaintext secrets even if the source `~/.claude.json` did — exporter halts with a clear error if it detects an inlined secret in the source config (per REQ-007)

**Task Breakdown Hint:**
- Task 2.1: Define canonical internal MCP schema (~3h)
- Task 2.2: Implement Claude→canonical reader, canonical→Claude/Cursor/Codex/opencode writers (~12h, mostly reusable from vsync prior art)
- Task 2.3: Implement env-var syntax translator (`${VAR}` ↔ `${env:VAR}` ↔ `${VAR:-default}`) (~3h)
- Task 2.4: Implement merge / collision logic per target (~5h)

**Dependencies:** REQ-001, REQ-003 (MCP format translation), REQ-007 (secrets contract)

---

### Story 3: Team lead publishes curated bundle for teammates

**As a** team lead who has curated a Claude Code setup that improves my team's productivity,
**I want to** publish a single agentpack bundle to a GitHub repo and have my team import it in one command,
**So that I can** standardize the team's AI tooling without writing a 5-page setup guide.

**Acceptance Criteria:**
- [ ] `agentpack export ~/.claude --out team-bundle.agentpack.tar.zst --strip-secrets --sign` produces a signed bundle with all `secret://` references intact but no resolved secret values
- [ ] `agentpack install https://github.com/myteam/agent-config/releases/latest/download/team-bundle.agentpack.tar.zst` fetches and installs from a remote URL (HTTPS + GitHub Releases supported in MVP)
- [ ] On install, agentpack verifies the bundle signature against a public key (provided via `--public-key <path>` or `--public-key-url <url>`)
- [ ] If the bundle references secret URIs (e.g. `secret://env/GITHUB_TOKEN`) and the local user has not configured them, install reports a clear list of unresolved secrets and exits non-zero
- [ ] User can run `agentpack secrets bind GITHUB_TOKEN env://GITHUB_TOKEN` to bind a local secret backend to the bundle's reference before re-attempting install
- [ ] Re-running `agentpack install` after `--update` flag updates only changed artifacts (incremental sync)

**Task Breakdown Hint:**
- Task 3.1: Implement bundle signing (minisign, ed25519) (~4h)
- Task 3.2: Implement bundle verification on install (~3h)
- Task 3.3: Implement HTTPS / GitHub Releases fetch (~3h)
- Task 3.4: Implement secrets binding subsystem (env backend MVP) (~6h)
- Task 3.5: Implement `--strip-secrets` lint mode on export (~3h)
- Task 3.6: Implement incremental install via content-hash comparison (~5h)

**Dependencies:** REQ-006 (bundle signing), REQ-007 (secrets contract), REQ-008 (remote fetch)

---

### Story 4: Personal cross-device sync via Git

**As a** developer with a laptop and a desktop where I want my Claude Code setup identical on both,
**I want to** commit my agentpack bundle to a private GitHub repo and pull it on either device,
**So that I can** treat my agent setup as a version-controlled artifact like my dotfiles.

**Acceptance Criteria:**
- [ ] `agentpack export ~/.claude --out ~/dotfiles/my-claude.agentpack.tar.zst --strip-secrets` produces a bundle safe to commit to git (no secrets)
- [ ] Workflow documented in README: edit on laptop → re-export → commit → push → pull on desktop → `agentpack install`
- [ ] Bundle filenames include a content-hash so users can see at a glance whether the bundle changed
- [ ] `agentpack diff old.tar.zst new.tar.zst` produces a human-readable diff (added/removed/changed artifacts)

**Task Breakdown Hint:**
- Task 4.1: Content-addressable bundle filename optional via `--hash-name` (~2h)
- Task 4.2: Implement `agentpack diff` command (~6h)

**Dependencies:** Story 3 acceptance criteria (signing optional here)

---

### Story 5: Bundle authors verify cross-agent compatibility before publishing

**As an** OSS skill/bundle author publishing for the community,
**I want to** know which target agents my bundle is actually compatible with before publishing,
**So that I can** declare compatibility honestly in the README and avoid issues from users with unsupported setups.

**Acceptance Criteria:**
- [ ] `agentpack lint my-bundle.agentpack.tar.zst` reports compatibility per target (Claude Code: ✅, Cursor: ⚠️ MCP only, Codex: ❌ depends on hook X which is not supported)
- [ ] Lint catches: inlined secrets (P0), invalid manifest schema (P0), missing referenced files (P0), hooks with no `os:` tag in a multi-OS bundle (P1), skill paths with absolute references (P1)
- [ ] Lint outputs JSON via `--json` for CI integration
- [ ] Lint exit code: 0 = clean, 1 = warnings, 2 = errors

**Task Breakdown Hint:**
- Task 5.1: Define lint rule registry (~3h)
- Task 5.2: Implement core lint rules (secrets, manifest schema, missing files) (~5h)
- Task 5.3: Implement per-target compatibility matrix (~4h)
- Task 5.4: JSON output mode (~2h)

**Dependencies:** REQ-002, REQ-007

---

## Functional Requirements

### Must Have (P0) — Critical for MVP Launch (Week 10)

#### REQ-001: Bundle format specification

**Description:** Define a canonical, versioned, deterministic bundle format that all agentpack operations produce and consume. The format must be reproducible (same input → same bytes), inspectable (human can `tar -tf` to see what's in it), and forward-compatible (older agentpack versions reject unknown spec versions cleanly).

**Acceptance Criteria:**
- [ ] Bundle is a `tar` archive compressed with `zstd` (level 19), file extension `.agentpack.tar.zst`
- [ ] Root of archive contains exactly: `manifest.yaml`, `artifacts/` (tree), and optionally `signatures/` (when signed)
- [ ] `manifest.yaml` schema versioned via top-level `agentpack: "1"` field
- [ ] Tar entries are emitted in sorted order with normalized mtime (epoch 0) and 0644/0755 mode for reproducibility
- [ ] SHA-256 content hash of bundle is deterministic given same input
- [ ] Bundle metadata includes: created-at, source-agent (e.g. `claude-code`), source-os (linux/darwin/win), agentpack-version

**Technical Specification:**

```yaml
# manifest.yaml schema (YAML 1.2, must validate against shipped JSON Schema)
agentpack: "1"
metadata:
  name: my-claude-setup
  version: "0.1.0"
  description: My personal Claude Code config
  authors:
    - "kaustubhupadhyaya"
  source_agent: claude-code
  source_os: linux            # linux | darwin | windows | wsl
  created_at: "2026-06-14T17:30:00+05:30"
  agentpack_version: "0.1.0"

# Compatibility declaration
targets:
  - claude-code               # primary
  - cursor                    # supported (MCPs only)
  - codex                     # supported (MCPs + commands)
  - opencode                  # supported

# Artifact index — paths relative to artifacts/ dir in tar
artifacts:
  skills:
    - path: skills/code-review/SKILL.md
      sha256: abc123...
    - path: skills/code-review/scripts/run.sh
      sha256: def456...
      mode: 0755
  slash_commands:
    - path: commands/ship.md
      sha256: 789abc...
  subagents:
    - path: agents/security-reviewer.md
      sha256: ...
  mcp_servers:
    - name: github
      transport: stdio
      command: npx
      args: ["-y", "@modelcontextprotocol/server-github"]
      env:
        GITHUB_PERSONAL_ACCESS_TOKEN: "secret://env/GITHUB_TOKEN"   # never inlined
    - name: postgres
      transport: stdio
      command: npx
      args: ["-y", "@modelcontextprotocol/server-postgres", "secret://env/PG_CONN_STRING"]

# Phase 2 fields (reserved, ignored in MVP)
hooks: []
permissions: {}
```

**Task Breakdown:**
- Define manifest YAML schema (`schemas/manifest.v1.json`): Small (3h)
- Implement YAML loader + JSON Schema validation: Small (3h)
- Implement reproducible tar+zstd writer (sorted, normalized mtime): Medium (5h)
- Implement reader / inspector (`agentpack inspect <bundle>`): Small (3h)
- Property-test reproducibility (same input → same hash): Small (3h)

**Dependencies:** None (foundational requirement)

---

#### REQ-002: `agentpack export` command — capture a Claude Code config to a bundle

**Description:** A single command that reads a Claude Code installation's user-config directory and produces a complete bundle. Must be safe by default (refuses to inline secrets) and idempotent (re-running with no changes produces the same bundle).

**Acceptance Criteria:**
- [ ] `agentpack export <claude-config-dir> [--out <path>] [--include <types>] [--strip-secrets] [--sign] [--dry-run]` is the supported signature
- [ ] Defaults: `--out` derives from manifest name + version + hash; `--include` defaults to all known types; `--strip-secrets` is implicit (cannot be disabled — refusing is a feature)
- [ ] Exporter reads from: `~/.claude/skills/`, `~/.claude/commands/`, `~/.claude/agents/`, `~/.claude/settings.json` (MCP entries only in MVP; hooks deferred to Phase 2), `~/.claude.json`, project `.mcp.json` if `--include-project` is passed
- [ ] On detecting an inlined secret value (heuristic: gitleaks regex + Shannon entropy + reserved field names like `apiKey`, `token`, `secret`), exporter halts with: artifact path, field, suggested replacement (`secret://env/<NAME>`)
- [ ] User can override with `--allow-inlined-secrets` (DANGEROUS, off by default, prints big warning)
- [ ] `--dry-run` prints a tree of artifacts that would be bundled, with sizes, without writing anything

**Technical Specification:**

```
# CLI
agentpack export ~/.claude \
    --out my-bundle.agentpack.tar.zst \
    --include skills,commands,subagents,mcp \
    --strip-secrets \
    --sign \
    --dry-run

# Pseudocode
fn export(cfg_dir, opts) -> Result<Bundle> {
    let scanner = ClaudeCodeScanner::new(cfg_dir);
    let artifacts = scanner.scan(opts.include)?;
    let secrets = SecretsScanner::new().scan_for_inlined(&artifacts)?;
    if !secrets.is_empty() && !opts.allow_inlined_secrets {
        return Err(Error::InlinedSecretsDetected(secrets));
    }
    let manifest = Manifest::build(&artifacts, &opts);
    let bundle = BundleWriter::new(opts.out).write(manifest, artifacts)?;
    if opts.sign {
        Signer::default().sign(&bundle, &opts.private_key)?;
    }
    Ok(bundle)
}
```

**Task Breakdown:**
- Claude Code config scanner (skills, commands, agents): Medium (6h)
- MCP entry reader from `~/.claude.json` + project `.mcp.json`: Small (4h)
- Inlined-secret detector (regex + entropy): Medium (5h)
- Export CLI command wiring + flag parsing: Small (3h)
- `--dry-run` tree printer: Small (2h)
- Integration test (export → inspect → assert): Small (3h)

**Dependencies:** REQ-001 (bundle format), REQ-007 (secrets contract)

---

#### REQ-003: `agentpack install` command — install a bundle into a target agent

**Description:** Reads a bundle from disk or HTTPS URL, verifies signature if present, resolves secret references against the local user's configured backends, and writes artifacts into the target agent's config directory using format-appropriate translations.

**Acceptance Criteria:**
- [ ] `agentpack install <bundle-source> [--target <agent>] [--into <config-dir>] [--on-collision <skip|overwrite|fail|backup>] [--dry-run]` is the supported signature
- [ ] `<bundle-source>` accepts: local path, `https://` URL, `github://owner/repo@tag` shorthand
- [ ] `--target` accepts: `claude-code` (default), `cursor`, `codex`, `opencode`
- [ ] Default collision policy: `backup` (renames existing file to `<file>.agentpack-backup-<timestamp>` and writes new)
- [ ] If bundle is signed and no `--public-key`/`--public-key-url` provided, install refuses with explanation (signed = expected to be verified)
- [ ] If bundle references unresolved secrets, install lists them and exits 1 with non-zero code
- [ ] Per-target translations (MVP): MCP entries → JSON for Claude/Cursor, JSONC for opencode, TOML for Codex; skills → identical content but normalize line endings to target OS convention; subagents → Claude only in MVP (warn on other targets)
- [ ] On success, prints summary: N skills installed, M MCP servers added, K commands installed, etc.

**Technical Specification:**

```
agentpack install my-bundle.agentpack.tar.zst --target codex
agentpack install https://github.com/myteam/cfg/releases/latest/download/team.tar.zst --target claude-code --on-collision skip
agentpack install github://myteam/cfg@v1.2.0 --target cursor --dry-run

# Per-target install logic (simplified)
match target {
    "claude-code" => ClaudeInstaller::new(into).install(bundle)?,
    "cursor"      => CursorInstaller::new(into).install(bundle)?,
    "codex"       => CodexInstaller::new(into).install(bundle)?,
    "opencode"    => OpencodeInstaller::new(into).install(bundle)?,
}
```

**Task Breakdown:**
- Bundle reader + signature verifier: Medium (5h)
- Remote-URL / github:// fetcher: Small (4h)
- Secrets resolver (env + keychain in MVP): Medium (5h)
- Per-target installer for Claude Code: Medium (6h)
- Per-target installer for Cursor: Small (4h)
- Per-target installer for Codex (TOML emit): Medium (5h)
- Per-target installer for opencode (JSONC emit, preserve comments): Medium (5h)
- Collision policy implementation: Small (3h)
- Install CLI wiring + summary output: Small (3h)
- Integration test matrix: install bundle → assert files: Medium (6h)

**Dependencies:** REQ-001, REQ-002, REQ-006, REQ-007

---

#### REQ-004: Cross-format MCP server translation

**Description:** Encode MCP server entries in a canonical internal schema, then emit per-target formats (JSON / JSONC / TOML) with correct env-var syntax per target.

**Acceptance Criteria:**
- [ ] Internal canonical MCP schema covers: `name`, `transport` (stdio / http / sse), `command` (for stdio), `args` (list), `env` (map, values may be secret URIs), `url` (for http/sse), `headers` (for http/sse)
- [ ] Round-trip test: `Claude (~/.claude.json) → canonical → Claude` is byte-identical for MCP fields
- [ ] Cross-target test: `Claude → canonical → Codex (~/.codex/config.toml [mcp_servers])` produces semantically equivalent TOML (assertable by re-parsing both)
- [ ] Env-var syntax translation: `${VAR}` (Claude/Cursor) ↔ `${env:VAR}` (opencode) ↔ `${VAR:-default}` (Codex literal expansion) handled symmetrically
- [ ] Unknown fields in source are preserved through round-trip when target is the same; on cross-target, unknown fields are logged as "lost in translation" warnings

**Technical Specification:**

```toml
# Codex emits (~/.codex/config.toml)
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { GITHUB_PERSONAL_ACCESS_TOKEN = "${GITHUB_TOKEN}" }
```

```json
// Claude emits (~/.claude.json fragment)
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

```jsonc
// opencode emits (~/.config/opencode/opencode.json)
{
  "mcp": {
    "github": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-github"],
      "environment": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${env:GITHUB_TOKEN}" }
    }
  }
}
```

**Task Breakdown:**
- Define canonical MCP schema (Rust struct / Go struct): Small (3h)
- Implement Claude reader/writer: Small (3h)
- Implement Codex TOML reader/writer (use `toml` crate): Small (3h)
- Implement opencode JSONC reader/writer (preserve comments via `jsonc-parser`): Medium (5h)
- Implement Cursor reader/writer (JSON, similar to Claude): Small (2h)
- Env-var syntax translation utility: Small (3h)
- Round-trip property tests: Medium (4h)

**Dependencies:** REQ-001

---

#### REQ-005: Secrets-as-URI contract (env + keychain backends in MVP)

**Description:** Secrets in bundles are stored as URI references like `secret://env/VAR_NAME` or `secret://keychain/MyService/MyAccount`. Bundle never contains plaintext secret values. On install, the local user's configured backend resolves the reference.

**Acceptance Criteria:**
- [ ] Secret URI grammar: `secret://<backend>/<path>` where `<backend>` ∈ {`env`, `keychain`} in MVP
- [ ] Resolver implements `env://`: read from process environment; if not set, return unresolved with helpful error
- [ ] Resolver implements `keychain://`: read from OS-native keychain — Windows DPAPI / Credential Manager, macOS Keychain, Linux libsecret (gnome-keyring / KWallet via Secret Service API)
- [ ] `agentpack secrets list` shows all secret URIs referenced in a bundle, with status (resolved / unresolved) per local backend config
- [ ] `agentpack secrets bind <name> <uri>` lets user rebind a reference (e.g. their secret is in keychain not env)
- [ ] Resolved secret values are written **only into the target agent's config** at install time; never logged, never echoed
- [ ] Round-trip test: source has `secret://env/X`, export bundle, install → target agent's config has `${X}` (NOT the resolved value) — install resolves lazily by emitting the env-var reference in the target agent's syntax. (Alternative mode `--resolve-on-install` is Phase 2.)

**Technical Specification:**

```
# Secret URI examples
secret://env/GITHUB_TOKEN
secret://keychain/com.anthropic.claude-code/github-pat

# Resolution
trait SecretBackend {
    fn name(&self) -> &str;
    fn resolve(&self, path: &str) -> Result<SecretValue>;
    fn has(&self, path: &str) -> bool;
}

struct EnvBackend;
impl SecretBackend for EnvBackend {
    fn resolve(&self, path: &str) -> Result<SecretValue> {
        std::env::var(path)
            .map(SecretValue::new)
            .map_err(|_| Error::Unresolved(format!("env://{path}")))
    }
}
```

**Task Breakdown:**
- Define `SecretBackend` trait + URI parser: Small (3h)
- Implement env backend: Trivial (1h)
- Implement Windows DPAPI keychain backend: Medium (5h) — use `windows-credentials` crate or equivalent
- Implement macOS Keychain backend: Medium (5h) — use `security-framework` crate or equivalent
- Implement Linux libsecret backend: Medium (5h) — use `secret-service` D-Bus API
- `agentpack secrets list / bind` CLI: Small (3h)
- Integration tests per OS in CI: Medium (5h)

**Dependencies:** REQ-001

---

#### REQ-006: Bundle signing and verification (minisign / ed25519)

**Description:** Bundles can be signed with a private ed25519 key (minisign format) and verified at install with a public key. Default in MVP is unsigned; signing is opt-in but recommended in docs.

**Acceptance Criteria:**
- [ ] `agentpack sign <bundle> --key <private-key>` produces a `.minisig` sidecar (or embedded signature in `signatures/` dir of the tar)
- [ ] `agentpack verify <bundle> --public-key <key>` returns 0 on valid signature, non-zero on tampered or wrong key, with clear error
- [ ] `agentpack install <signed-bundle>` automatically verifies if signature present; requires `--public-key` or `--public-key-url` (HTTPS only) — never trusts an embedded key without explicit user opt-in
- [ ] Verification covers bundle bytes + manifest hash; tampering with any file in bundle invalidates signature
- [ ] `agentpack keygen` produces a fresh ed25519 keypair, prints public key, prompts user to securely store private key

**Technical Specification:**

```
# Signing
agentpack keygen --out ~/agentpack-keys/
agentpack sign team.agentpack.tar.zst --key ~/agentpack-keys/private.key

# Verifying
agentpack verify team.agentpack.tar.zst --public-key ~/agentpack-keys/public.key
agentpack install team.agentpack.tar.zst --public-key-url https://team.example.com/agentpack.pub
```

**Task Breakdown:**
- Pick crate: `minisign-verify` + `ed25519-dalek` (Rust) or equivalent: Small (1h research)
- Implement `agentpack keygen`: Small (2h)
- Implement `agentpack sign`: Small (3h)
- Implement `agentpack verify`: Small (3h)
- Wire into install command (auto-verify if signature present): Small (2h)
- Tamper-detection test (modify bundle, verify fails): Small (2h)

**Dependencies:** REQ-001

---

#### REQ-007: Inlined-secret detection on export (gitleaks-style)

**Description:** Before producing a bundle, agentpack scans source config for likely-secret values (long base64-ish strings, known patterns like AWS access keys, GitHub tokens, etc.) and refuses to bundle by default.

**Acceptance Criteria:**
- [ ] Detection rules cover: AWS access keys (`AKIA[0-9A-Z]{16}`), GitHub PATs (`ghp_[A-Za-z0-9]{36}`), generic high-entropy strings (≥40 chars, Shannon entropy ≥ 4.5), known service-specific patterns (Slack, Stripe, OpenAI keys)
- [ ] Detector runs against values inside `env:` maps in MCP configs, value strings in `settings.json`, and any file contents in skills
- [ ] False-positive controls: pattern allowlist file `~/.config/agentpack/secrets-allowlist.txt`; per-rule `--ignore <rule-id>` flag
- [ ] On detection, exporter halts with: rule ID, artifact path, line/field, snippet (masked), suggested replacement (`secret://env/<INFERRED_NAME>`)
- [ ] Exit code 2 (distinct from generic error 1) for secrets-detected case (CI-friendly)

**Task Breakdown:**
- Import or vendor a gitleaks-rules-like ruleset: Small (3h)
- Implement Shannon entropy calculator + threshold tunable: Small (2h)
- Hook detector into export pipeline: Small (3h)
- Build allowlist + per-rule ignore mechanism: Small (3h)
- Test fixture: known-bad config asserts export refusal: Small (2h)

**Dependencies:** None (independent module; integrates with REQ-002)

---

#### REQ-008: `agentpack wsl-sync` — one-shot Windows ↔ WSL bridge

**Description:** A specialized subcommand that copies Claude Code's `~/.claude/` between Windows-native and WSL-native filesystems on the same machine, with normalization. Solves the user's stated personal pain.

**Acceptance Criteria:**
- [ ] `agentpack wsl-sync --from win-native --to wsl --distro Ubuntu` copies `C:\Users\<u>\.claude\` to `\\wsl$\Ubuntu\home\<u>\.claude\` (paths auto-detected, user can override)
- [ ] `agentpack wsl-sync --from wsl --to win-native --distro Ubuntu` does the reverse
- [ ] On Win→WSL: CRLF → LF conversion for text files (`.md`, `.sh`, `.py`); executable bit set on `.sh` and `.py` per shebang detection; `.DS_Store` / `Thumbs.db` excluded
- [ ] On WSL→Win: LF → CRLF conversion optional via `--line-endings crlf|lf|preserve` (default: preserve)
- [ ] OneDrive collision detection: if `C:\Users\<u>\.claude\` lives inside an OneDrive-synced folder, command warns and offers to exclude
- [ ] `--dry-run` prints planned copies + transformations without writing
- [ ] Runs **only** on Windows hosts (errors clearly on Linux/macOS — "use `agentpack install` instead")
- [ ] Idempotent: re-running with no changes produces no writes

**Technical Specification:**

```
# Common use
agentpack wsl-sync --from win-native --to wsl              # auto-detects distro
agentpack wsl-sync --from wsl --to win-native --distro Ubuntu --dry-run

# Implementation outline
1. Resolve source path (Win: $env:USERPROFILE\.claude; WSL: \\wsl$\<distro>\home\<user>\.claude)
2. Resolve dest path
3. Walk source, build copy plan with transformations annotated
4. If --dry-run, print plan and exit
5. Otherwise execute: copy, transform line endings, set exec bits via WSL chmod
6. Print summary: N files copied, M transformed
```

**Task Breakdown:**
- Detect WSL distros on Windows host (`wsl.exe -l -v` parse): Small (3h)
- Path translation Win ↔ `\\wsl$\<distro>\...`: Small (2h)
- File walker with ignore patterns: Small (2h)
- CRLF↔LF transformer (binary detection to avoid mangling): Small (3h)
- Executable-bit setter via `wsl.exe chmod`: Small (2h)
- OneDrive path detection + warning: Small (3h)
- `--dry-run` planner: Small (2h)
- E2E test on Windows runner: Medium (5h)

**Dependencies:** None (standalone subcommand, reuses generic file utilities)

---

#### REQ-009: `agentpack lint` — pre-flight bundle and config validation

**Description:** Standalone validator that checks a bundle (or a live config directory) against a rule registry. Used by `export` internally (gate before writing) and by users / CI for verification.

**Acceptance Criteria:**
- [ ] `agentpack lint <bundle-or-dir>` runs all P0 rules and exits 0/1/2 (clean/warnings/errors)
- [ ] `--json` outputs machine-readable results for CI
- [ ] `--rules <list>` runs only specified rules
- [ ] Rule registry includes (MVP): `inlined-secret`, `invalid-manifest-schema`, `missing-referenced-file`, `unsupported-target-artifact`, `non-portable-absolute-path` (any path starting with `C:\` or `/home/specific-user/`)
- [ ] Each rule has: ID, severity (error / warning), description, fix suggestion

**Task Breakdown:**
- Rule trait + registry: Small (3h)
- Implement 5 MVP rules: Medium (6h)
- CLI wiring + JSON output: Small (3h)
- Test fixtures (good + each-rule-bad): Small (4h)

**Dependencies:** REQ-001, REQ-007

---

### Should Have (P1) — Important but Not Blocking

#### REQ-010: Phase 2 — hooks support with OS-tag filtering

**Description:** Extend the bundle format and installer to handle hooks, with each hook tagged with `os:` constraints so that a multi-OS bundle ships hooks-per-OS rather than trying to translate shell commands.

**Acceptance Criteria:**
- [ ] Manifest schema v2 adds `hooks:` array with entries `{event, os, command, matcher}`
- [ ] Exporter reads hooks from Claude Code `settings.json`; tags them with the source OS
- [ ] Installer skips hooks whose `os:` doesn't match the local OS, logs decisions
- [ ] Backward compatible with v1 bundles (older bundles install fine, just no hooks)

**Task Breakdown:**
- Bump manifest schema to v2 (additive): Small (2h)
- Hook reader from Claude `settings.json`: Small (3h)
- Hook installer per target agent: Medium (5h)
- OS-tag filter logic + tests: Small (3h)

**Dependencies:** REQ-001, REQ-003 (Phase 1 must be stable)

---

#### REQ-011: Phase 2 — additional secret backends (age, sops, 1Password CLI)

**Description:** Add resolver implementations for `age://`, `sops://`, and `op://` URIs so bundles can target diverse user setups.

**Acceptance Criteria:**
- [ ] `secret://age/<file>?path=<dot.notation>` resolves an age-encrypted YAML/JSON
- [ ] `secret://sops/<file>?path=...` resolves a sops-encrypted file
- [ ] `secret://op/<vault>/<item>/<field>` resolves via `op` CLI (1Password)
- [ ] Backends are pluggable; each requires user-side setup documented in `docs/secrets-backends.md`

**Dependencies:** REQ-005 (trait must exist)

---

#### REQ-012: Phase 2 — permissions/allowlist translation across agents

**Description:** Translate Claude Code's `settings.json` `permissions` block to Cursor/Codex/opencode equivalents.

**Acceptance Criteria:**
- [ ] Per-agent translation tables maintained in `translation/permissions/`
- [ ] Lossy translations logged (e.g. Claude's `ask` rule has no Codex equivalent → log warning)
- [ ] User can opt out per agent via `--skip-permissions`

**Dependencies:** REQ-003

---

#### REQ-013: Phase 2 — git-remote-based cross-device sync

**Description:** Add `agentpack sync` command that pushes/pulls a bundle to/from a configured git remote, with content-hash-based change detection.

**Acceptance Criteria:**
- [ ] `agentpack sync init <git-remote-url>` records the remote in `~/.config/agentpack/sync.yaml`
- [ ] `agentpack sync push` re-exports current config, compares to last-pushed hash, pushes only if changed
- [ ] `agentpack sync pull` fetches latest bundle, runs install if hash differs
- [ ] Optional: `agentpack sync watch` runs push on filesystem-change (Phase 2.5)

**Dependencies:** REQ-002, REQ-003

---

#### REQ-014: Phase 2 — team bundle re-bind UX

**Description:** When a teammate imports your bundle, secrets resolve against *their* backends, not yours. The re-bind UX is an interactive prompt that walks them through each secret reference.

**Acceptance Criteria:**
- [ ] On `install` with `--interactive`, agentpack prompts for each unresolved secret: "Bundle requires `secret://env/GITHUB_TOKEN`. Bind to (env / keychain / age / sops / skip): "
- [ ] Bindings are saved per-bundle to `~/.config/agentpack/bindings/<bundle-name>.yaml` so subsequent installs of the same bundle don't re-prompt

**Dependencies:** REQ-005

---

### Nice to Have (P2) — Phase 3 / Future

#### REQ-015: Phase 3 — bidirectional sync with 3-way merge

**Description:** Two users (or one user across devices) make independent changes to their configs; agentpack reconciles via a lockfile and 3-way merge using git semantics.

**Dependencies:** Significant; gated on Phase 1+2 stability and real-user feedback.

#### REQ-016: Phase 3 — live Windows ↔ WSL daemon

**Description:** Background process watching both `C:\Users\<u>\.claude\` and `\\wsl$\<distro>\home\<u>\.claude\` propagating changes both ways with conflict resolution.

**Dependencies:** REQ-008 (one-shot must be solid first)

#### REQ-017: Phase 3 — chezmoi adapter

**Description:** `agentpack export --as-chezmoi-template` emits chezmoi-flavored templates with secret backend hooks, for users on the chezmoi-based dotfiles workflow.

**Dependencies:** REQ-001, REQ-005

---

## Non-Functional Requirements

### Performance

**Command execution time (one-time, not throughput):**
- `agentpack export` for typical 30-artifact config: **< 2 seconds** (p95) on a mid-range laptop
- `agentpack install` from local bundle for same setup: **< 3 seconds** (p95)
- `agentpack install` from HTTPS URL adds network time; tool-time budget excluding network is < 3 seconds
- `agentpack wsl-sync` for 30-artifact config across the 9P boundary: **< 30 seconds** (p95) — slowness is filesystem-bound, not tool-bound

**Resource Usage:**
- Memory: < 100 MB peak during export/install (no streaming buffer should exceed this — typical bundles are MB-scale)
- Disk: bundle compression ratio target ≥ 3× over raw artifact size for typical text-heavy configs

**Reproducibility:**
- Exporting the same source twice in a row produces byte-identical bundles (SHA-256 match) within 1 minute window
- Subsecond CPU difference between two consecutive exports of the same source

---

### Security

**Bundle integrity:**
- Bundles are content-addressable by SHA-256 of the tar+zstd bytes
- Optional ed25519 signatures via minisign format (MVP); cosign-keyless support is Phase 2 nice-to-have
- Verification on install: failed signature → abort install (exit 3)

**Secrets handling (the differentiator):**
- **Inlined secrets are rejected by default** at export — bundle cannot ship plaintext keys without explicit user override
- All secret references in bundles use the `secret://` URI scheme; plaintext never enters the bundle byte stream
- Resolved secrets are written **only into target agent's config at install time**; never logged at any verbosity level, never echoed in `--dry-run` output, never included in error messages (masked instead)
- Allowlist mechanism for false-positive secret detection: `~/.config/agentpack/secrets-allowlist.txt`

**Tool authentication / supply chain:**
- Release binaries are signed via GitHub Actions OIDC + minisign
- Published install instructions verify binary signature before executing
- Reproducible builds (Go: `-trimpath -buildvcs=false`) so binaries can be independently rebuilt from source
- Dependency review on PRs (Dependabot + `nancy` or equivalent)

**Network / data:**
- `agentpack install <https-url>` only accepts HTTPS, refuses HTTP (exit 1)
- No telemetry collected (privacy by design)
- No external network calls during export / install / lint unless explicitly fetching a remote bundle

---

### Reliability

**Failure modes:**
- All file writes (install, sign) are atomic: write to `<file>.tmp`, fsync, rename
- Partial-install on error: rollback by restoring `.agentpack-backup-*` files; transactionally documented in `~/.config/agentpack/log/<timestamp>.log`
- Bundle reading is fault-tolerant: corrupted zstd → clear error; corrupted manifest YAML → clear error pointing to schema violation

**Exit codes (CI-friendly):**
- 0 = success
- 1 = generic error
- 2 = secrets-detected (export refused)
- 3 = signature verification failed
- 4 = unresolved secrets (install needs `secrets bind`)
- 5 = unsupported target / artifact for given operation

**Error messages:**
- Every error message includes: what went wrong, suggested next action, exit code
- No raw stack traces in user-facing errors unless `--verbose` / `RUST_LOG=debug` (or `AGENTPACK_DEBUG=1`)

---

### Compatibility

**Target operating systems (MVP):**
- Windows 10 1809+ / Windows 11 (x86_64, ARM64)
- macOS 12+ (x86_64, ARM64 / Apple Silicon)
- Linux (x86_64, ARM64) — Ubuntu 22.04+, Debian 12+, Fedora 39+ (glibc ≥ 2.31)
- WSL2 with Ubuntu / Debian / Fedora distros (full support); WSL1 unsupported

**Target agents (MVP):**
- Claude Code (Anthropic) — primary, full support
- OpenAI Codex CLI — MCPs + slash commands
- opencode (sst) — MCPs + skills
- Cursor — MCPs

**Target agents (Phase 2):**
- GitHub Copilot CLI (MCPs)
- Gemini CLI (instructions + MCPs)
- Cline / Windsurf (rule formats — best effort)

**Bundle format forward compatibility:**
- Schema versioned via `agentpack: "1"` field; future versions add fields, never remove or repurpose
- Older agentpack binaries reading v2 bundles MUST refuse cleanly with "spec version not supported, please upgrade"

**Tool runtime:**
- Single static binary per platform; no Node.js / Python / Go runtime required at install or runtime
- Go 1.22+ at build time (target Go for single-binary distribution per decision in [Open Questions / Q1](#q1-language-choice))

---

### Observability

**Logging:**
- Default log level: INFO (one-line summaries per operation)
- `--verbose` / `-v` → DEBUG (per-artifact actions)
- `-vv` → TRACE (file-level operations)
- `--quiet` → WARN+ only
- Log destination: stderr by default; `--log-file <path>` redirects

**Operation manifests:**
- Every `install` writes a log entry to `~/.config/agentpack/log/<timestamp>-install.log` containing: bundle path, bundle hash, target, files written, files backed up, errors
- Logs are plain text + JSON sidecar for tooling

---

## Technical Considerations

### System Architecture

**Greenfield project**, written in **Go** (decision rationale in [Open Questions / Q1](#q1-language-choice)). Single static binary distribution.

**Component diagram:**

```
                       ┌──────────────────────────────────────┐
                       │           agentpack CLI              │
                       │ (urfave/cli for command parsing)     │
                       └────────────────┬─────────────────────┘
                                        │
       ┌────────────────────────────────┼─────────────────────────────────┐
       │                                │                                 │
       v                                v                                 v
┌──────────────┐                ┌──────────────┐                   ┌──────────────┐
│   scanner    │                │   bundle     │                   │  installers  │
│ Reads source │                │  Manifest +  │                   │ Per-agent:   │
│ agent config │                │  tar+zstd    │                   │ Claude/Cursor│
│              │                │  marshaller  │                   │ /Codex/      │
│              │                │              │                   │  opencode    │
└──────┬───────┘                └──────┬───────┘                   └──────┬───────┘
       │                               │                                  │
       └───────────┬───────────────────┴──────────────────────┬───────────┘
                   │                                          │
                   v                                          v
            ┌──────────────┐                          ┌──────────────────┐
            │   linter     │                          │   secrets        │
            │ Rule registry│                          │ backend trait +  │
            │ (secrets,    │                          │ env/keychain/age │
            │ schema, ...) │                          │   impls          │
            └──────────────┘                          └──────────────────┘
                                                              │
                                                              v
                                                     ┌─────────────────┐
                                                     │ Per-OS keychain │
                                                     │ Win DPAPI       │
                                                     │ macOS Keychain  │
                                                     │ Linux libsecret │
                                                     └─────────────────┘
```

**Module boundaries:**

| Module | Responsibility | Major deps |
|---|---|---|
| `cmd/agentpack` | CLI entrypoint, flag parsing | `urfave/cli/v3` |
| `internal/bundle` | Format reader/writer, manifest schema | `gopkg.in/yaml.v3`, `github.com/klauspost/compress/zstd`, `archive/tar` |
| `internal/scanner` | Reads source agent config | filesystem |
| `internal/installer/claude` | Writes to Claude Code config | `internal/bundle` |
| `internal/installer/codex` | Writes to Codex config | `github.com/pelletier/go-toml/v2` |
| `internal/installer/opencode` | Writes to opencode config | `github.com/tidwall/jsonc` |
| `internal/installer/cursor` | Writes to Cursor config | `encoding/json` |
| `internal/secrets` | URI parser, backend trait, env+keychain impls | OS-native bindings |
| `internal/secrets/keychain_windows.go` | DPAPI / Credential Manager | `github.com/danieljoos/wincred` |
| `internal/secrets/keychain_darwin.go` | macOS Keychain | `github.com/keybase/go-keychain` |
| `internal/secrets/keychain_linux.go` | libsecret via D-Bus | `github.com/zalando/go-keyring` |
| `internal/linter` | Lint rules + registry | `internal/bundle` |
| `internal/sign` | minisign signing/verifying | `github.com/jedisct1/go-minisign` |
| `internal/wsl` | WSL detection + bridge | OS calls (`wsl.exe -l -v`) |

---

### Manifest Schema (canonical, full)

JSON Schema (excerpt; full file ships in `schemas/manifest.v1.json`):

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://agentpack.dev/schema/manifest.v1.json",
  "type": "object",
  "required": ["agentpack", "metadata"],
  "properties": {
    "agentpack": { "const": "1" },
    "metadata": {
      "type": "object",
      "required": ["name", "version", "source_agent", "created_at", "agentpack_version"],
      "properties": {
        "name": { "type": "string", "pattern": "^[a-z0-9][a-z0-9-]{0,62}$" },
        "version": { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+(?:-[a-zA-Z0-9.-]+)?$" },
        "description": { "type": "string", "maxLength": 280 },
        "authors": { "type": "array", "items": { "type": "string" } },
        "source_agent": { "enum": ["claude-code", "cursor", "codex", "opencode", "other"] },
        "source_os": { "enum": ["linux", "darwin", "windows", "wsl"] },
        "created_at": { "type": "string", "format": "date-time" },
        "agentpack_version": { "type": "string" }
      }
    },
    "targets": {
      "type": "array",
      "items": { "enum": ["claude-code", "cursor", "codex", "opencode"] }
    },
    "artifacts": {
      "type": "object",
      "properties": {
        "skills": { "type": "array" },
        "slash_commands": { "type": "array" },
        "subagents": { "type": "array" },
        "mcp_servers": { "type": "array" }
      }
    }
  }
}
```

---

### Secrets URI Grammar

```
secret-uri  := "secret://" backend "/" path [ "?" query ]
backend     := "env" | "keychain" | "age" | "sops" | "op"            # MVP: env, keychain
path        := segment ("/" segment)*
segment     := unreserved-char+
query       := key "=" value ("&" key "=" value)*
```

Examples:
```
secret://env/GITHUB_TOKEN
secret://keychain/com.agentpack.default/github-pat
secret://age/secrets.age?path=mcp.github.token        # Phase 2
secret://sops/secrets.enc.yaml?path=mcp.github.token  # Phase 2
secret://op/Private/GitHub PAT/credential             # Phase 2
```

---

### Database Schema

**N/A** — agentpack is a stateless CLI. All state is on-disk:
- `~/.config/agentpack/` (Linux/macOS) or `%APPDATA%\agentpack\` (Windows) for user config (secrets-allowlist, sync remote, key bindings)
- `~/.config/agentpack/log/` for operation logs
- No database, no daemon, no background process in MVP (Phase 3 only)

---

### External Dependencies

**Third-Party Services:**

1. **GitHub Releases** (for distribution + remote bundle fetch via `github://` URLs):
   - Purpose: hosting bundles for `github://owner/repo@tag` shorthand resolution
   - API: public GitHub REST API
   - Rate limits: 60 req/hour unauthenticated, 5000 authenticated (if user opts in)
   - Fallback: direct HTTPS URL if GitHub unavailable

2. **MCP Registry** ([modelcontextprotocol.io](https://blog.modelcontextprotocol.io/posts/2025-09-08-mcp-registry-preview/)):
   - Purpose: optional metadata enrichment for MCP server entries (Phase 2)
   - Integration: opt-in, not required for MVP

**Internal Dependencies (Go libraries):**
- `urfave/cli/v3` — CLI parsing
- `klauspost/compress/zstd` — zstd codec (pure Go)
- `gopkg.in/yaml.v3` — YAML parsing
- `pelletier/go-toml/v2` — TOML for Codex
- `tidwall/jsonc` — JSONC comment-preserving parser for opencode
- `jedisct1/go-minisign` — bundle signing
- `danieljoos/wincred` — Windows Credential Manager (build-tag `windows`)
- `keybase/go-keychain` — macOS Keychain (build-tag `darwin`)
- `zalando/go-keyring` — Linux libsecret (build-tag `linux`)
- `gitleaks/go-gitleaks-config` — secrets ruleset (vendored, not online)

---

### Migration Strategy

**This is a greenfield tool — no migration from prior versions in MVP. Forward-compatibility plan:**

1. **Bundle format versioning:** Top-level `agentpack: "<version>"` field; v1 → v2 transitions are additive (new optional fields). Breaking changes bump to v2 and ship a `agentpack migrate v1-to-v2` command.

2. **CLI flag compatibility:** Stable subcommands from MVP (`export`, `install`, `lint`, `verify`, `sign`, `keygen`, `secrets`, `wsl-sync`, `inspect`, `diff`) will not change signatures across 0.x → 1.0. New subcommands may be added. Deprecation cycle: deprecate in 0.x → 1, remove in 0.x+2.

3. **User config compatibility:** `~/.config/agentpack/*` files are versioned via a `version:` field; older binaries reading newer config refuse with clear upgrade message.

---

### Testing Strategy

**Unit Tests:**
- Coverage target: **≥ 80%** for `internal/bundle`, `internal/secrets`, `internal/linter`
- Tools: standard `go test` + `github.com/stretchr/testify` for assertions
- Key areas:
  - Bundle round-trip (write → read → assert equality)
  - Manifest schema validation
  - Secret URI parser (round-trip, malformed input)
  - MCP format translation matrix (all 4 agents × all 3 formats)

**Integration Tests:**
- Per-OS suite running in GitHub Actions on `windows-latest`, `macos-latest`, `ubuntu-latest`
- Tests:
  - Full export → bundle → install round-trip with assertion of byte-identical artifact content
  - Cross-agent install: Claude → bundle → Codex install → assert TOML structurally equivalent
  - Secret resolution against env backend (set env in CI, install, assert resolved)
  - Bundle signing → tamper → verify fails

**E2E Tests:**
- Test fixtures in `tests/fixtures/`:
  - `claude-30-artifacts/` — realistic 30-artifact source
  - `bad-inlined-secret/` — config with inlined GitHub PAT (must be refused)
  - `multi-os-hooks/` — bundle with hooks for linux + windows (Phase 2)
- Each fixture has expected outputs + golden bundle hash

**Performance Tests:**
- Benchmarks in `bench/` covering export + install for typical-config-size (30 artifacts)
- CI gate: regression > 50% on benchmark median triggers failure

**Security Tests:**
- Fuzzer for manifest YAML loader (`go-fuzz` or `go test -fuzz`)
- Bundle parser fuzzed against random corrupt zstd / corrupt tar
- Path-traversal protection: assert that bundles cannot write outside target dir (i.e. no `../../../etc/passwd` in tar entry names)

**Cross-platform CI matrix:**
- 3 OSes × {build, unit, integration} × {x86_64, ARM64 where feasible}
- Total: ~12 CI jobs per PR

---

## Implementation Roadmap

### Phase 1: MVP Foundation (Weeks 1–10)

#### Week 1–2: Bundle format + CLI scaffold

**Goal:** Get the foundational primitives down — manifest, tar+zstd writer/reader, CLI skeleton.

**Tasks:**
- [ ] **Task 1.1:** Initialize Go module, set up linting (`golangci-lint`), set up GitHub Actions CI matrix (Win/Mac/Linux × build+test)
  - Complexity: Small (3h)
  - Dependencies: None
- [ ] **Task 1.2:** Define `manifest.v1.json` JSON Schema; implement YAML loader + validator (`internal/bundle/manifest`)
  - Complexity: Medium (5h)
  - Dependencies: 1.1
- [ ] **Task 1.3:** Implement reproducible tar+zstd bundle writer (sorted entries, normalized mtime); add property test
  - Complexity: Medium (5h)
  - Dependencies: 1.2
- [ ] **Task 1.4:** Implement bundle reader + `agentpack inspect <bundle>` command
  - Complexity: Small (4h)
  - Dependencies: 1.3
- [ ] **Task 1.5:** Set up `urfave/cli/v3` skeleton with subcommand stubs (`export`, `install`, `lint`, etc.)
  - Complexity: Small (2h)
  - Dependencies: 1.1

**Validation Checkpoint 1:** Can `inspect` a hand-written bundle; reproducible build of an empty bundle produces deterministic SHA-256

---

#### Week 3–4: Export + Claude Code scanner

**Goal:** Capture a real Claude Code config into a bundle.

**Tasks:**
- [ ] **Task 2.1:** Implement Claude Code scanner (`internal/scanner/claude`) — reads `~/.claude/skills/`, `~/.claude/commands/`, `~/.claude/agents/`, `~/.claude.json` MCP entries, project `.mcp.json`
  - Complexity: Medium (6h)
  - Dependencies: 1.4
- [ ] **Task 2.2:** Implement inlined-secret detector (gitleaks rules + entropy)
  - Complexity: Medium (5h)
  - Dependencies: 1.3
- [ ] **Task 2.3:** Wire `agentpack export <dir>` end-to-end (scan → secret-check → bundle-write)
  - Complexity: Small (3h)
  - Dependencies: 2.1, 2.2
- [ ] **Task 2.4:** Implement `--dry-run` planner with tree printer
  - Complexity: Small (2h)
  - Dependencies: 2.3
- [ ] **Task 2.5:** Integration test: real Claude Code fixture → export → inspect, assert content
  - Complexity: Small (4h)
  - Dependencies: 2.3

**Validation Checkpoint 2:** `agentpack export ~/.claude` on a real config produces an inspectable, deterministic bundle; halts on inlined secret

---

#### Week 5–6: Install + per-agent translators

**Goal:** Install bundles into target agents, including format translation.

**Tasks:**
- [ ] **Task 3.1:** Define canonical internal MCP schema; implement Claude reader/writer
  - Complexity: Small (4h)
  - Dependencies: 1.2
- [ ] **Task 3.2:** Implement Codex TOML reader/writer (env-var syntax conversion)
  - Complexity: Medium (5h)
  - Dependencies: 3.1
- [ ] **Task 3.3:** Implement opencode JSONC reader/writer (comment preservation)
  - Complexity: Medium (5h)
  - Dependencies: 3.1
- [ ] **Task 3.4:** Implement Cursor JSON reader/writer
  - Complexity: Small (3h)
  - Dependencies: 3.1
- [ ] **Task 3.5:** Implement Claude installer (writes skills/commands/agents to filesystem, MCPs to settings)
  - Complexity: Medium (6h)
  - Dependencies: 3.1
- [ ] **Task 3.6:** Implement Codex installer (TOML writes only — Codex has no skills)
  - Complexity: Small (4h)
  - Dependencies: 3.2
- [ ] **Task 3.7:** Implement opencode + Cursor installers
  - Complexity: Medium (6h)
  - Dependencies: 3.3, 3.4
- [ ] **Task 3.8:** Implement collision policy (`skip / overwrite / fail / backup`)
  - Complexity: Small (3h)
  - Dependencies: 3.5
- [ ] **Task 3.9:** Wire `agentpack install <bundle> --target <agent>` end-to-end
  - Complexity: Small (3h)
  - Dependencies: 3.5–3.8

**Validation Checkpoint 3:** Full round-trip: export from Claude → install into Codex → verify MCP entries appear correctly in TOML

---

#### Week 7: Secrets resolution + keychain backends

**Goal:** Implement the secrets-aware bundle promise — env + keychain backends.

**Tasks:**
- [ ] **Task 4.1:** Define `SecretBackend` trait + URI parser
  - Complexity: Small (3h)
  - Dependencies: 1.2
- [ ] **Task 4.2:** Implement env backend
  - Complexity: Trivial (1h)
  - Dependencies: 4.1
- [ ] **Task 4.3:** Implement Windows DPAPI / Credential Manager backend (`internal/secrets/keychain_windows.go`)
  - Complexity: Medium (5h)
  - Dependencies: 4.1
- [ ] **Task 4.4:** Implement macOS Keychain backend (`internal/secrets/keychain_darwin.go`)
  - Complexity: Medium (5h)
  - Dependencies: 4.1
- [ ] **Task 4.5:** Implement Linux libsecret backend (`internal/secrets/keychain_linux.go`)
  - Complexity: Medium (5h)
  - Dependencies: 4.1
- [ ] **Task 4.6:** Implement `agentpack secrets list / bind` commands
  - Complexity: Small (3h)
  - Dependencies: 4.1
- [ ] **Task 4.7:** Wire secret resolution into install pipeline (lazy: emit env-var refs in target config)
  - Complexity: Small (4h)
  - Dependencies: 4.1, 3.9

**Validation Checkpoint 4:** Bundle with `secret://env/X` installs into target with `${X}`; bundle with `secret://keychain/...` resolves and inserts at install time correctly on all 3 OSes

---

#### Week 8: Signing + WSL bridge

**Goal:** Ship signing and the user's stated personal pain solution.

**Tasks:**
- [ ] **Task 5.1:** Implement `agentpack keygen` (ed25519 via minisign)
  - Complexity: Small (2h)
  - Dependencies: 1.3
- [ ] **Task 5.2:** Implement `agentpack sign / verify`
  - Complexity: Small (4h)
  - Dependencies: 5.1
- [ ] **Task 5.3:** Wire install-time signature verification (auto if signature present)
  - Complexity: Small (3h)
  - Dependencies: 5.2, 3.9
- [ ] **Task 5.4:** Implement WSL detection (`wsl.exe -l -v` parsing) — Win-only build tag
  - Complexity: Small (3h)
  - Dependencies: None
- [ ] **Task 5.5:** Implement path translation Win ↔ `\\wsl$\<distro>\...`
  - Complexity: Small (2h)
  - Dependencies: 5.4
- [ ] **Task 5.6:** Implement CRLF↔LF transformer with binary-detection
  - Complexity: Small (3h)
  - Dependencies: None
- [ ] **Task 5.7:** Wire `agentpack wsl-sync` command with `--dry-run`, `--from`, `--to`
  - Complexity: Medium (5h)
  - Dependencies: 5.4, 5.5, 5.6
- [ ] **Task 5.8:** OneDrive collision detection + warning
  - Complexity: Small (3h)
  - Dependencies: 5.4

**Validation Checkpoint 5:** Sign a bundle, verify, tamper → verify fails. `agentpack wsl-sync` produces correct copy with line-ending normalization on real Windows+WSL host.

---

#### Week 9: Linter + remote bundle fetch + polish

**Goal:** Round out user-visible functionality + remote install support.

**Tasks:**
- [ ] **Task 6.1:** Define lint rule registry trait
  - Complexity: Small (3h)
  - Dependencies: 1.4
- [ ] **Task 6.2:** Implement MVP rules (`inlined-secret`, `invalid-manifest-schema`, `missing-referenced-file`, `unsupported-target-artifact`, `non-portable-absolute-path`)
  - Complexity: Medium (6h)
  - Dependencies: 6.1
- [ ] **Task 6.3:** Implement `agentpack lint <bundle-or-dir>` with `--json` output
  - Complexity: Small (3h)
  - Dependencies: 6.2
- [ ] **Task 6.4:** Implement `github://owner/repo@tag` URL fetcher
  - Complexity: Small (4h)
  - Dependencies: 3.9
- [ ] **Task 6.5:** Implement direct HTTPS fetch with content-hash verification
  - Complexity: Small (3h)
  - Dependencies: 6.4
- [ ] **Task 6.6:** Implement `agentpack diff <bundle-a> <bundle-b>` command
  - Complexity: Medium (5h)
  - Dependencies: 1.4

**Validation Checkpoint 6:** `agentpack install github://kaustubh/my-config@v0.1.0 --target claude-code` works end-to-end; lint catches all known-bad fixtures.

---

#### Week 10: Release prep + benchmarks + docs

**Goal:** Ship a usable 0.1.0.

**Tasks:**
- [ ] **Task 7.1:** Build release pipeline in GitHub Actions: cross-compile for win-x64, win-arm64, darwin-x64, darwin-arm64, linux-x64, linux-arm64
  - Complexity: Medium (5h)
  - Dependencies: None
- [ ] **Task 7.2:** Sign release binaries via GitHub OIDC + minisign
  - Complexity: Medium (5h)
  - Dependencies: 7.1, 5.2
- [ ] **Task 7.3:** Write `benchmarks/ttrc.md` — measured TTRC against fixture; assert Goal 1 target
  - Complexity: Small (4h)
  - Dependencies: All install/export
- [ ] **Task 7.4:** Write README.md (install, quickstart, examples)
  - Complexity: Medium (6h)
  - Dependencies: All MVP features
- [ ] **Task 7.5:** Write `docs/bundle-format.md` (canonical spec)
  - Complexity: Small (4h)
  - Dependencies: 1.2
- [ ] **Task 7.6:** Write `docs/secrets.md` (URI scheme + backend setup per OS)
  - Complexity: Small (4h)
  - Dependencies: 4.6
- [ ] **Task 7.7:** Cut 0.1.0 release; publish binaries to GitHub Releases; submit to Claude Code Plugin marketplace as `agentpack`
  - Complexity: Small (3h)
  - Dependencies: 7.1, 7.2, 7.4

**Validation Checkpoint 7 (MVP launch):** Binary installable on all 3 OSes; quickstart in README works end-to-end; TTRC benchmark meets Goal 1 target (< 5 min).

---

### Phase 2: Hooks + Cross-Device (Weeks 11–16, post-MVP)

**Roadmap-level, not yet detailed-task-broken:**

- Hooks support (manifest v2, OS-tag filtering)
- Permissions/allowlist translation across agents
- `agentpack sync init/push/pull` command + `~/.config/agentpack/sync.yaml`
- Team-bundle re-bind interactive UX
- Secret backends: age, sops, 1Password CLI

**Estimated effort:** ~60–80 hours; ~6 weeks part-time.

---

### Phase 3: Bidirectional Sync + Live Daemons (Months 4–6+)

**Roadmap-level only:**

- Bidirectional sync with lockfile + 3-way merge (`agentpack sync watch`)
- Live Windows↔WSL daemon (replaces one-shot `wsl-sync`)
- Custom subagent support for non-Claude agents (where they exist)
- chezmoi adapter (emit chezmoi templates)
- Cosign keyless signing as alternative to minisign

**Gated on:** Phase 1+2 user feedback, validated demand, measurable Goal 5 progress.

---

### Task Dependencies Visualization

```
Week 1-2 (Foundation):
  1.1 (Module + CI) → 1.2 (Manifest) → 1.3 (Bundle Writer) → 1.4 (Reader/Inspect)
                  → 1.5 (CLI Scaffold)

Week 3-4 (Export):
  1.3 → 2.2 (Secret Detect)
  1.4 → 2.1 (Claude Scanner) → 2.3 (Export Wire) → 2.4 (Dry-Run) → 2.5 (Integ Test)
                            ↑
                        2.2 ─┘

Week 5-6 (Install + Translation):
  1.2 → 3.1 (Canonical MCP) → 3.2 (Codex) & 3.3 (opencode) & 3.4 (Cursor)
  3.1 → 3.5 (Claude Install)
  3.2 → 3.6 (Codex Install)
  3.3, 3.4 → 3.7 (opencode/Cursor Install)
  3.5 → 3.8 (Collision) → 3.9 (Install Wire)

Week 7 (Secrets):
  1.2 → 4.1 (Trait) → 4.2 (env) & 4.3 (Win) & 4.4 (Mac) & 4.5 (Linux) → 4.6 (CLI) → 4.7 (Install Wire)

Week 8 (Signing + WSL):
  1.3 → 5.1 (keygen) → 5.2 (sign/verify) → 5.3 (install verify)
  5.4 (WSL Detect) → 5.5 (Paths) → 5.7 (wsl-sync wire) ← 5.6 (CRLF)
                                  ↑
                              5.8 (OneDrive) ┘

Week 9 (Lint + Remote):
  1.4 → 6.1 → 6.2 → 6.3 (Lint Wire)
  3.9 → 6.4 (github://) → 6.5 (HTTPS)
  1.4 → 6.6 (diff)

Week 10 (Release):
  All MVP → 7.1 (Cross-compile) → 7.2 (Sign Releases) → 7.7 (Ship)
                                                       ↑
                              7.3 (Benchmarks) & 7.4 (README) & 7.5 + 7.6 (Docs)

Critical Path: 1.1 → 1.2 → 1.3 → 2.1 → 2.3 → 3.1 → 3.5 → 4.1 → 4.7 → 5.7 → 7.1 → 7.7
```

---

### Effort Estimation

**Phase 1 Total Estimated Effort:**
- Week 1–2 (Foundation): ~19 hours
- Week 3–4 (Export): ~20 hours
- Week 5–6 (Install + Translators): ~39 hours
- Week 7 (Secrets): ~26 hours
- Week 8 (Signing + WSL): ~25 hours
- Week 9 (Lint + Remote): ~24 hours
- Week 10 (Release): ~31 hours
- **Total: ~184 hours** (~6 weeks full-time / ~9–10 weeks at ~20h/week part-time)

**Risk Buffer:** +25% (46 hours) for unknowns, especially OS-specific keychain bugs and WSL edge cases
**Final Phase 1 Estimate:** ~230 hours (~7–8 weeks full-time / ~11–12 weeks part-time)

**Phase 2:** ~70 hours estimated, +25% buffer = ~90 hours
**Phase 3:** unestimated; gated on Phase 2 outcomes.

---

## Out of Scope

Explicitly NOT included in MVP (Week 10) or Phase 2 (Week 16):

1. **Hosted SaaS / cloud sync**
   - Reason: Operationally expensive; trust model is hard (secrets in transit); enterprise product, not solo-dev fit
   - Future: Possibly Phase 4 with explicit demand signal

2. **GUI / desktop app**
   - Reason: CLI covers all listed user stories; GUI would expand scope without expanding fit
   - Future: Consider after 1000+ active CLI users

3. **Plugin marketplace / discovery layer of our own**
   - Reason: Wouldn't be additive — Claude Code Plugin marketplace already exists, MCP Registry exists. We integrate with them.
   - Future: `agentpack search` (list available bundles from configured sources) is plausible Phase 2.5 if demanded

4. **Cross-translation of hook shell commands (bash ↔ PowerShell)**
   - Reason: Open-ended translation problem; high-risk for correctness; users surface their own bugs.
   - Workaround: Phase 2 ships `os:` tags so multi-OS bundles ship one hook per OS

5. **Claude.ai cloud connectors**
   - Reason: Not filesystem-accessible; no public API; out of scope of "file-based config sync"

6. **Telemetry / usage analytics**
   - Reason: Privacy by design; not worth the trust cost for solo-dev tool

7. **Cosign keyless signing in MVP**
   - Reason: Sigstore-dependent (extra trust assumption); minisign is simpler and self-contained
   - Future: Phase 2/3 as optional alternative

8. **Per-target *deep* subagent translation (Cursor/Codex agents)**
   - Reason: Subagent concept varies wildly across agents; Phase 1 supports Claude-only subagents; cross-agent is Phase 3+ research

9. **MCP server *content* validation** (does the server actually work)
   - Reason: agentpack handles configuration sync, not server testing. Recommend `mcp inspector` for that.

10. **Bundle encryption** (encrypting the whole bundle for confidential team sharing)
    - Reason: Out of scope for MVP; users with this need can `age`-encrypt the bundle externally.
    - Future: Phase 3 consideration

---

## Open Questions & Risks

### Open Questions

#### Q1: Language choice

- **Current Status:** Defaulted to **Go** in PRD (single static binary, matches skillshare distribution model, mature OS-keychain libraries, fast compile)
- **Options:** (A) Go — recommended (B) Rust — safer types, but slower iteration and harder cross-compile (C) Node/TS — npm-distributable but Node runtime dependency
- **Owner:** kaustubh (builder)
- **Deadline:** **Week 1** (blocks everything)
- **Impact:** High (all dependencies and tooling flow from this)
- **Recommendation rationale:** skillshare (the closest active competitor at 2.2k★) chose Go and ships single binaries successfully. Go is the lowest-risk choice given the user is learning end-to-end code writing. Rust adds learning overhead; Node adds runtime overhead.

#### Q2: Signing scheme

- **Current Status:** Defaulted to **minisign / ed25519** for MVP (simpler, self-contained, no Sigstore trust dependency)
- **Options:** (A) minisign — recommended for MVP (B) cosign keyless — Sigstore-backed, but adds dependency (C) Both — minisign in MVP, cosign added in Phase 2
- **Owner:** kaustubh
- **Deadline:** **Week 8** (when signing is implemented)
- **Impact:** Medium (additive — doesn't block other work)
- **Recommendation:** Ship minisign in MVP. Add cosign Phase 2 if demanded.

#### Q3: Manifest format

- **Current Status:** Defaulted to **YAML** (human-friendly, supports comments)
- **Options:** (A) YAML — recommended (B) TOML — matches Codex but less expressive (C) JSON — universal but no comments
- **Owner:** kaustubh
- **Deadline:** **Week 1**
- **Impact:** Medium (downstream tooling assumes parser)
- **Recommendation:** YAML with strict JSON Schema validation. Comments are useful for hand-authored bundles.

#### Q4: Default install collision policy

- **Current Status:** Defaulted to **backup** (rename existing file, write new)
- **Options:** (A) backup (B) skip-on-collision (C) overwrite (D) fail
- **Owner:** kaustubh
- **Deadline:** Week 5
- **Impact:** Low (configurable; default is just first impression)
- **Recommendation:** backup is safest. Cheaper to be wrong-but-recoverable than wrong-and-lost.

#### Q5: Bundle URL shorthand syntax

- **Current Status:** Defaulted to `github://owner/repo@tag`
- **Options:** (A) `github://` and `gitlab://` (B) Plain `git+https://` (C) Skip shorthand, require full HTTPS
- **Owner:** kaustubh
- **Deadline:** Week 9
- **Impact:** Low
- **Recommendation:** Ship `github://` only in MVP. Add `gitlab://` when first user asks.

---

### Risks & Mitigation

(Pulled from research red-team and elaborated.)

| Risk | Likelihood | Impact | Severity | Mitigation | Contingency |
|---|---|---|---|---|---|
| **Anthropic / OpenAI ships cross-agent plugin format, obsoletes agentpack** | Medium | High | **High** | Bet on standards convergence taking 18+ months (AGENTS.md precedent); position as adapter not transformer; integrate with Claude Code Plugin marketplace from MVP so we are *additive* | Pivot to "secrets-aware bundle layer ON TOP of Claude Code Plugin format" if format wins |
| **Skillshare adds MCP support, becomes complete competitor** | High | Medium | **Medium** | Differentiate on secrets-as-URI and Windows↔WSL — neither is on skillshare's roadmap as of v0.20.14 | Contribute MCP+secrets PRs to skillshare if appropriate, focus agentpack on WSL/signing/bundle-format niche |
| **OS-keychain libraries break on niche Linux distros (NixOS, Alpine without libsecret)** | Medium | Medium | **Medium** | Test on Ubuntu / Fedora / Debian in CI; document workarounds; provide `env://` fallback always available | Ship `--no-keychain` flag; users use age/sops in Phase 2 |
| **CRLF↔LF transformer corrupts a binary file mistaken for text** | Low | High | **High** | Robust binary detection (null-byte sniff + extension allowlist for transformation); never transform files whose source extension is not in `.md`, `.sh`, `.py`, `.js`, `.ts`, `.txt` | Bug + patch; clear restore path via backup files |
| **Signed bundle key compromise (user's private key leaked)** | Low | High | **High** | Document key-rotation best practice in README; recommend hardware key storage; key never logged | User revokes by changing the verifier public key on installer side |
| **Path-traversal in tar entries (zip-slip equivalent)** | Low | Critical | **High** | Whitelist relative paths only in reader; reject any `..`, absolute paths, or unusual names | Fuzz testing in CI; security advisory if found in wild |
| **Format-translation bug silently corrupts user's working MCP config** | Medium | High | **High** | `--dry-run` default-first in docs; automatic backup on install; round-trip property tests in CI | Restore from `.agentpack-backup-*` files |
| **WSL bridge tested only on Ubuntu, breaks on Debian/Fedora WSL** | Medium | Medium | **Medium** | Test on at least 2 distros in CI matrix | Document tested distros; PR welcome for others |
| **Solo-dev burnout / scope creep blowing past Week 10** | High | Medium | **Medium** | Strict phasing — Phase 1 features only; defer any Phase 2 temptation explicitly; weekly self-review against this PRD | Ship a smaller MVP (drop signing → Phase 2) if Week 8 looks blocked |
| **Low adoption (< 100 stars in 30 days)** | Medium | Medium | **Medium** | Cross-post launch to r/ClaudeAI, r/cursor, HN; submit to Claude Code Plugin marketplace; ship a real fixture-driven demo video | Iterate on positioning; consider deeper Windows↔WSL angle as standalone hook for visibility |

---

## Validation Checkpoints

(All seven inline above per-week; consolidated here for at-a-glance.)

### Checkpoint 1 (End of Week 2): Foundation
**Criteria:**
- [ ] `agentpack inspect <bundle>` works on hand-written test bundle
- [ ] Two consecutive bundle builds of the same input produce identical SHA-256
- [ ] CI passes on Win/Mac/Linux for build + unit tests

**If Failed:** Rework bundle format primitives before continuing to export logic

---

### Checkpoint 2 (End of Week 4): Export pipeline
**Criteria:**
- [ ] `agentpack export ~/.claude` on real Claude Code config produces valid bundle
- [ ] Inlined-secret detector refuses fixture with known bad GitHub PAT (exit 2)
- [ ] `--dry-run` prints accurate tree without writes

**If Failed:** Either secrets detector is too noisy/quiet or scanner misses artifact types — fix before install work

---

### Checkpoint 3 (End of Week 6): Cross-agent install
**Criteria:**
- [ ] Round-trip Claude → bundle → Claude is byte-identical for skills/commands/agents
- [ ] Cross-agent Claude → bundle → Codex produces valid `~/.codex/config.toml` with MCP entries
- [ ] Collision policy `backup` is default; preserves prior content with timestamped suffix

**If Failed:** Translation correctness is foundational — don't move on to secrets until this passes

---

### Checkpoint 4 (End of Week 7): Secrets on all 3 OSes
**Criteria:**
- [ ] env backend works on all 3 OSes (trivial but tested)
- [ ] Keychain backend works on Win (DPAPI), Mac (Keychain), Linux (libsecret) — write + read round-trip
- [ ] Bundle with `secret://env/X` installs into Claude with `${X}` in `~/.claude.json` MCP entry

**If Failed:** Likely a per-OS keychain library issue — investigate, swap library if needed, do not ship without all 3

---

### Checkpoint 5 (End of Week 8): Signing + WSL bridge
**Criteria:**
- [ ] Signed bundle verifies with correct public key; verification fails after tampering
- [ ] `agentpack wsl-sync` (on Windows host) correctly copies skills from Win-native to WSL-Ubuntu with CRLF→LF
- [ ] OneDrive warning fires when source path is under OneDrive

**If Failed:** Signing is independent — if blocked, defer to Phase 2 and ship MVP unsigned. WSL bridge is core to user pain — debug seriously.

---

### Checkpoint 6 (End of Week 9): Lint + remote bundle install
**Criteria:**
- [ ] `agentpack lint` catches all 5 MVP rules on known-bad fixtures
- [ ] `agentpack install github://kaustubh/test-bundle@v0.0.1` fetches + installs end-to-end
- [ ] `agentpack diff a.tar.zst b.tar.zst` produces readable output

**If Failed:** Lint shipping late is acceptable (gate by P0 rules only); remote fetch is critical for team-sharing story

---

### Checkpoint 7 (Week 10 — MVP launch):
**Criteria:**
- [ ] Cross-compiled binaries for win-x64, win-arm64, darwin-x64, darwin-arm64, linux-x64, linux-arm64 published on GitHub Releases
- [ ] Release binaries signed via OIDC + minisign; signature verifiable from README instructions
- [ ] README quickstart (`brew install` / `winget install` / `curl ... | sh` → export → install) works on fresh machines
- [ ] TTRC benchmark documented; meets Goal 1 (< 5 min for 30-artifact fixture)
- [ ] Submission to Claude Code Plugin marketplace accepted

**If Failed (per-criterion):**
- Binary issues → ship for 3 OSes you can support, ARM later
- Quickstart broken → fix before tweeting
- TTRC missed → publish honest number anyway and aim for improvement next release

---

## Appendix: Task Breakdown Hints

### Suggested TaskMaster Task Structure

**Phase 1 Foundation Tasks (Week 1–2, ~19h):**
1. Initialize Go module + linting + GitHub Actions CI matrix (Small, 3h)
2. Define `manifest.v1.json` JSON Schema + YAML loader + validator (Medium, 5h)
3. Implement reproducible tar+zstd bundle writer (Medium, 5h)
4. Implement bundle reader + `agentpack inspect` (Small, 4h)
5. Set up `urfave/cli` skeleton with subcommand stubs (Small, 2h)

**Export & Scanner Tasks (Week 3–4, ~20h):**
6. Implement Claude Code scanner (skills/commands/agents/MCPs) (Medium, 6h)
7. Implement inlined-secret detector (gitleaks + entropy) (Medium, 5h)
8. Wire `agentpack export <dir>` end-to-end (Small, 3h)
9. Implement `--dry-run` planner (Small, 2h)
10. Integration test: real fixture → export → inspect (Small, 4h)

**Translation & Install Tasks (Week 5–6, ~39h):**
11. Define canonical internal MCP schema + Claude reader/writer (Small, 4h)
12. Codex TOML reader/writer + env-var conversion (Medium, 5h)
13. opencode JSONC reader/writer (comment preserving) (Medium, 5h)
14. Cursor JSON reader/writer (Small, 3h)
15. Claude installer (Medium, 6h)
16. Codex installer (Small, 4h)
17. opencode + Cursor installers (Medium, 6h)
18. Collision policy implementation (Small, 3h)
19. Wire `agentpack install` end-to-end (Small, 3h)

**Secrets Tasks (Week 7, ~26h):**
20. SecretBackend trait + URI parser (Small, 3h)
21. env backend (Trivial, 1h)
22. Windows DPAPI keychain backend (Medium, 5h)
23. macOS Keychain backend (Medium, 5h)
24. Linux libsecret backend (Medium, 5h)
25. `agentpack secrets list / bind` CLI (Small, 3h)
26. Wire secret resolution into install (Small, 4h)

**Signing & WSL Tasks (Week 8, ~25h):**
27. `agentpack keygen` (Small, 2h)
28. `agentpack sign / verify` (Small, 4h)
29. Install-time signature verification (Small, 3h)
30. WSL detection (`wsl.exe -l -v` parsing) (Small, 3h)
31. Win ↔ WSL path translation (Small, 2h)
32. CRLF↔LF transformer with binary detection (Small, 3h)
33. Wire `agentpack wsl-sync` command (Medium, 5h)
34. OneDrive collision detection (Small, 3h)

**Lint & Remote Tasks (Week 9, ~24h):**
35. Lint rule registry trait (Small, 3h)
36. Implement 5 MVP lint rules (Medium, 6h)
37. `agentpack lint` CLI with `--json` (Small, 3h)
38. `github://owner/repo@tag` URL fetcher (Small, 4h)
39. HTTPS fetcher with content-hash verification (Small, 3h)
40. `agentpack diff` command (Medium, 5h)

**Release & Docs Tasks (Week 10, ~31h):**
41. Cross-compile release pipeline (Medium, 5h)
42. Sign release binaries via OIDC + minisign (Medium, 5h)
43. TTRC benchmark + `benchmarks/ttrc.md` (Small, 4h)
44. Write README.md (install, quickstart, examples) (Medium, 6h)
45. Write `docs/bundle-format.md` (Small, 4h)
46. Write `docs/secrets.md` (Small, 4h)
47. Cut 0.1.0 release; submit to Claude Code Plugin marketplace (Small, 3h)

**USER-TEST checkpoint tasks (insert at week boundaries):**
- USER-TEST 1 (after Task 5): manually inspect a hand-built bundle
- USER-TEST 2 (after Task 10): export real Claude Code config, verify content
- USER-TEST 3 (after Task 19): cross-agent install round-trip on user's own setup
- USER-TEST 4 (after Task 26): verify keychain backend works on user's actual Mac/Win/Linux
- USER-TEST 5 (after Task 34): WSL bridge on user's actual Win+WSL machine — the headline personal-pain test
- USER-TEST 6 (after Task 40): install from `github://` URL of a real bundle
- USER-TEST 7 (after Task 47): full quickstart on a fresh machine

**Total Phase 1: ~47 implementation tasks + 7 USER-TEST tasks = 54 tasks; ~184 hours unbuffered; ~230 hours buffered.**

### Parallelizable Tasks

**Can work in parallel:**
- Tasks 11–14 (translators) once 11 done — 12, 13, 14 are independent
- Tasks 22, 23, 24 (per-OS keychain backends) — independent
- Tasks 41 (cross-compile) and 44 (README) — independent
- Task 6 (scanner) and Task 7 (secret detector) — independent

**Must be sequential:**
- 1 → 2 → 3 (foundation chain)
- 6/7 → 8 → 10 (export chain)
- 11 → 15 (Claude install needs canonical schema)
- 20 → 22/23/24 → 26 (secrets chain)

### Critical Path Tasks

1. Task 1 (Module setup)
2. Task 2 (Manifest schema)
3. Task 3 (Bundle writer)
4. Task 6 (Claude scanner)
5. Task 8 (Export wire)
6. Task 11 (MCP canonical + Claude r/w)
7. Task 15 (Claude installer)
8. Task 19 (Install wire)
9. Task 20 (Secrets trait)
10. Task 26 (Install + secrets wire)
11. Task 33 (wsl-sync wire)
12. Task 41 (cross-compile)
13. Task 47 (release)

**Critical path duration:** ~70 hours = ~9 weeks at 8h/week, ~3.5 weeks full-time. Buffered: ~12 weeks part-time.

---

**End of PRD**

*This PRD is optimized for TaskMaster AI task generation. All P0 requirements include task breakdown hints, complexity estimates (Trivial / Small / Medium / Large with hour ranges), and dependency mapping. USER-TEST checkpoints insert at every ~5–7 implementation tasks per the prd-taskmaster skill convention. Source research and red-team analysis are linked at the top of this document.*
