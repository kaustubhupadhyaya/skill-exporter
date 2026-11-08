# Skill Exporter: Deep Research Brief

**Researcher:** Claude (deep-research skill)
**Date:** 2026-06-14
**Scope:** Cross-environment portability of AI agent tooling (skills, MCP servers, hooks, slash commands, custom agents, plugins, connectors) across four sync boundaries.

---

## TL;DR

1. **The problem is real, validated, and the market is moving.** GitGuardian found 24,008 unique secrets exposed in MCP configs on public GitHub in 2026, with AI-assisted commits leaking at 2× the baseline rate — proof that developers are actively wrestling with sharing/syncing these configs in unsafe ways. Multiple sync tools have shipped in the last 12 months ([vsync](https://github.com/nicepkg/vsync), [sync-agents-settings](https://github.com/Leoyang183/sync-agents-settings), [skillshare](https://github.com/runkids/skillshare), [antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills)) — the user is not first, but the space is unconsolidated.

2. **No existing tool covers the full surface area.** Every direct competitor handles a subset: vsync handles MCPs+skills+agents+commands but no hooks; skillshare (2.2k★) handles skills+agents+rules but no MCPs; sync-agents-settings handles MCPs+instructions only. **No tool covers MCPs + skills + hooks + slash commands + agents simultaneously**, and **no tool natively bridges the Windows ↔ WSL filesystem split** that is the user's stated personal pain.

3. **Standards are converging on instructions but fragmenting on everything else.** [AGENTS.md](https://addozhang.medium.com/agents-md-a-new-standard-for-unified-coding-agent-instructions-0635fc5cb759) (Linux Foundation, 30+ agents reading it) has won the "instructions" battle. The [MCP Registry](https://blog.modelcontextprotocol.io/posts/2025-09-08-mcp-registry-preview/) (~2,000 servers, Linux Foundation as of Dec 2025) has won the "discovery" battle. **But local config sync, hooks, skills, and slash-command portability remain per-agent zoos with no standard in sight.**

4. **The Build-vs-Buy verdict is "Build, but small and complementary."** Forking skillshare or vsync gets you 70% of where they already are. The defensible wedge is in the gaps: (a) a sealed/signed bundle format with first-class secrets handling, (b) the Windows↔WSL bridge, and (c) bidirectional sync with merge semantics. None of these are solved.

5. **Strongest risk: the platforms eat the niche.** Claude Code's plugin marketplace (`/plugin install` from any git URL) is 90% of the "team sharing" job already, and the format is open. If Anthropic + OpenAI + Cursor agree on a shared plugin manifest within 12 months, generic sync tools become wrappers. Bet on this consolidating; build adapters not transformers.

**Overall confidence: HIGH** on competitor landscape and pain validation, **MEDIUM** on adoption trajectory of any new entrant given the crowded field, **LOW** on whether AGENTS.md-style cross-vendor coordination will extend to skills/MCPs within 12-18 months.

---

## Context & Scope

The user — a developer hitting the Windows-installed Claude Code skills not appearing in WSL-side Claude Code on the same machine — wants to build a "skill exporter" that handles smooth transitions across:

1. **OS boundary** (same device): Windows ↔ WSL ↔ macOS ↔ Linux VM ↔ Docker dev containers.
2. **Agent boundary** (same device): Claude Code ↔ OpenAI Codex CLI ↔ opencode ↔ Cursor ↔ Cline ↔ Continue.dev ↔ Windsurf.
3. **Cross-device** (same agent, multiple machines): laptop ↔ desktop ↔ remote dev server.
4. **Team sharing**: export a curated "skill pack" or "agent bundle" a teammate can import in one command.

**Frameworks applied:**
- **Competitive Positioning Map** (Layer 2 / 3) — plot existing tools on coverage × maturity × distribution model
- **Build-vs-Buy Matrix** (Layer 3, Decision Points) — evaluate forking vs new-build
- **Jobs-To-Be-Done** (Layer 3) — three distinct personas with distinct underlying jobs
- **Analogous Industry Scan** (Layer 4) — VSCode/JetBrains cross-IDE sync, chezmoi-style dotfiles, npm/cargo package manager portability

---

## The Landscape

### Concrete config file locations (the unglamorous baseline)

This is the substrate any sync tool has to operate on. **Every agent stores config differently**, and that fragmentation IS the problem.

| Agent | User config root | MCP config | Skills dir | Format | Notes |
|---|---|---|---|---|---|
| **Claude Code** | `~/.claude/` (Unix/WSL) `C:\Users\<u>\.claude\` (Win native) | `~/.claude.json` + `.mcp.json` (project) | `~/.claude/skills/<name>/SKILL.md` | JSON | `settings.json` for hooks/perms; `settings.local.json` for non-VCS overrides; `.claude-plugin/plugin.json` for plugins |
| **OpenAI Codex CLI** | `~/.codex/` | `~/.codex/config.toml` `[mcp_servers]` | n/a (no skill concept) | **TOML** | Override path via `CODEX_HOME` env var; project-scoped `.codex/config.toml` in trusted projects only |
| **opencode** | `~/.config/opencode/opencode.json(c)` | same file, `mcp` key | `~/.config/opencode/skills/` | **JSONC** (JSON w/ comments) | Project config traverses up to git root; remote orgs can serve `.well-known/opencode` |
| **Cursor** | `~/.cursor/` | `~/.cursor/mcp.json` | uses extensions, not "skills" | JSON | Has built-in settings sync (cloud) |
| **GitHub Copilot** | `~/.copilot/` | `~/.copilot/mcp-config.json` | n/a | JSON | |
| **Cline / Windsurf** | per-IDE storage | proprietary rule formats | proprietary | JSON-ish | Resist normalization |
| **Gemini CLI** | varies | `~/.gemini/` or similar | reads AGENTS.md / GEMINI.md | JSON | |

The three format families (JSON, JSONC, TOML) and the four env-var syntaxes (`${VAR}` / `${env:VAR}` / `{env:VAR}` / `$VAR`) are why naive copy-paste fails. Existing sync tools spend most of their LOC on these conversions.

### The Claude Code surface area, fully enumerated

A "skill exporter" worth shipping must decide which of these to handle:

| Element | Lives where | Format | Bundleable today? |
|---|---|---|---|
| Skills | `~/.claude/skills/<n>/SKILL.md` + assets | Markdown + YAML frontmatter | ✅ portable, AGENTS.md-adjacent |
| MCP servers | `~/.claude.json`, `.mcp.json` | JSON | ✅ portable, MCP Registry-adjacent |
| Hooks | `~/.claude/settings.json` `hooks:` key | JSON (shell commands inside) | ⚠️ OS-specific commands |
| Slash commands | `~/.claude/commands/<n>.md` + project `.claude/commands/` | Markdown | ✅ portable |
| Custom agents (subagents) | `~/.claude/agents/<n>.md` | Markdown + frontmatter | ✅ portable |
| Permissions / `allow`/`deny` rules | `settings.json` | JSON | ⚠️ project-sensitive |
| Plugins | `.claude-plugin/plugin.json` manifest | JSON manifest pointing to everything above | ✅ **the convergence format** |
| Connectors (Claude.ai-side) | cloud-side | n/a | ❌ not file-system at all |
| Marketplaces | `.claude-plugin/marketplace.json` in a git repo | JSON | ✅ team-distribution primitive |
| Statusline / output styles | `settings.json` | JSON | ✅ portable |

**Key observation:** Claude Code's `.claude-plugin/plugin.json` format already bundles skills + MCPs + hooks + commands into one installable unit. It's the closest thing to a canonical bundle the ecosystem has. But it's Claude-specific — Codex/Cursor/opencode don't read it.

### Existing solutions audit

| Tool | Stars | Scope | What it does | What it doesn't | Status |
|---|---|---|---|---|---|
| **[skillshare](https://github.com/runkids/skillshare)** | 2.2k | Skills + agents + rules + commands across 60+ AI tools | Go binary, Git-based distribution, NTFS junction support on Windows, security audits | **No MCP sync, no hooks**, no built-in secrets mgmt | Active, v0.20.14 Jun 2026 |
| **[vsync](https://github.com/nicepkg/vsync)** | 51 | MCPs + skills + agents + commands across Claude/Cursor/OpenCode/Codex | JSON↔JSONC↔TOML conversion, env-var syntax translation, manifest-based diff | **No hooks**, no Windows/WSL bridge, no secrets, one-way only | Active, v1.2, roadmap mentions watch mode |
| **[sync-agents-settings](https://github.com/Leoyang183/sync-agents-settings)** | 9 | MCPs + instructions only, 12+ agents | TypeScript, format-specific writers, backup before write | **No skills, no hooks**, no merge (skips existing), one-way (Claude→others) | Active, v0.4.3 Mar 2026 |
| **[antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills)** | 40.6k | Curated 1,500+ skill library w/ `npx` installer | Pull-based distribution, multi-tool targets (Claude/Cursor/Codex/Gemini/Antigravity/Kiro/opencode/Copilot), bundles | **Not a sync tool** — it's a content marketplace + installer. No personal-bundle export. | Very active |
| **[claude-skills (alirezarezvani)](https://github.com/alirezarezvani/claude-skills)** | 18k | 337 skills + 30+ agents content library | Multi-agent compatible (12+ tools) | Content not sync | Active |
| **[Claude Code Plugins / Marketplace](https://code.claude.com/docs/en/plugins-reference)** | (official) | Full Claude surface (skills+MCPs+hooks+cmds) | `.claude-plugin/plugin.json` manifest, `/plugin install <git-url>`, official JSON schemas at schemastore.org | **Claude Code only**, no cross-agent, no secrets sealing | Active, growing |
| **[ccpi](https://github.com/jeremylongshore/claude-code-plugins-plus-skills)** | (mid-size) | 425 plugins, 2,810 skills, 200 agents | CLI package manager for Claude plugin marketplace | Claude-only | Active |
| **[mcp-installer (anaisbetts)](https://github.com/anaisbetts/mcp-installer)** | mid | MCP server installation across clients | An MCP server that installs MCP servers (meta) | Not a sync tool | Active |
| **[mcp-installer (joobisb)](https://github.com/joobisb/mcp-installer)** | mid | MCP install across Claude/Cursor/Gemini/etc. | CLI installer | Per-server, not bulk-sync | Active |
| **[add-mcp (neondatabase)](https://github.com/neondatabase/add-mcp)** | mid | `npx add-mcp <name>` cross-agent install | Single-server, multi-target install | Not bidirectional, not bundle-aware | Active |
| **[mcphub](https://www.mcphub.app/)** | mid | MCP server discovery + GUI installer | Desktop app + CLI, registry | Discovery not sync | Active |
| **[get-mcp-keys](https://github.com/StuMason/get-mcp-keys)** | small | Secrets loader for Cursor MCP | Loads keys from home dir, keeps out of repos | Just loading, not sync | Active |
| **[Chezmoi-based patterns](https://dev.to/dotwee/one-skills-brain-for-codex-claude-cursor-and-copilot-with-chezmoi-2p3k)** | (chezmoi=12k★) | Whatever you template | Cross-device, cross-agent via templates, age-encrypted secrets | **Manual per-user templating**, no out-of-box AI knowledge | Mature dotfiles base |
| **[MCP Registry (official)](https://blog.modelcontextprotocol.io/posts/2025-09-08-mcp-registry-preview/)** | (official) | Discovery layer | Canonical metadata, namespace verification, ~2,000 servers | **Discovery only** — does not sync local config | Live, governed by Linux Foundation |
| **[AGENTS.md spec](https://www.morphllm.com/agents-md-guide)** | (standard) | Instruction file portability | 30+ agents read it natively, Linux Foundation governance | **Instructions only** — not skills/MCPs/hooks | Adopted standard |
| **Cursor cloud Settings Sync** | (built-in) | Extensions, settings, keybindings | Cross-device for Cursor specifically | Cursor only | Live |

### Competitive positioning: the coverage gap

Plotted on a coverage matrix (✅ supported, ⚠️ partial, ❌ missing):

| | MCPs | Skills | Hooks | Slash Cmds | Subagents | Windows↔WSL | Secrets-Aware | Bidirectional | Team Bundle Format |
|---|---|---|---|---|---|---|---|---|---|
| vsync | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ | ⚠️ env-only | ❌ one-way | ❌ |
| sync-agents-settings | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ⚠️ env-only | ❌ one-way | ❌ |
| skillshare | ❌ | ✅ | ❌ | ⚠️ | ✅ | ⚠️ NTFS only | ❌ detect-only | ⚠️ via git | ⚠️ via git |
| antigravity-awesome-skills | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | n/a | ❌ pull-only | ✅ npx bundles |
| chezmoi pattern | ✅ | ✅ | ✅ | ✅ | ✅ | ⚠️ manual | ✅ via age/sops | ✅ via git | ⚠️ manual |
| Claude Code Plugins | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ install-only | ✅ marketplace.json |
| **THE GAP** | | | | | | **all-❌** | **all-❌/⚠️** | | |

The bottom row tells you exactly where the differentiation room is.

---

## Implementation Reality: What Practitioners Actually Experience

This is the depth layer — the gap between marketing and operational reality.

### The Windows-WSL split is a real, named, unresolved pain

The user's stated personal pain is concretely documented:

> "Skills installed in WSL are only available when running Claude Code from WSL. Skills installed in the Windows path are only available when running Claude Code from PowerShell or Command Prompt. These are two separate file systems."
> — [agensi.io guide](https://www.agensi.io/learn/claude-code-skills-windows-complete-guide)

The Anthropic GitHub repo has an open feature request, [Issue #49933](https://github.com/anthropics/claude-code/issues/49933): "Feature Request: Native WSL Remote Integration for Claude Code Desktop on Windows." The issue explicitly enumerates and **rejects** the obvious workarounds:

- "Accessing `\\wsl.localhost\...` from Windows" — works but slow + path mismatches
- "Running Claude Code entirely inside WSL via WSLg" — adds friction
- "Manual CLI inside WSL + separate Windows UI" — fragmented workflow, two disconnected surfaces

**Concrete gotchas catalogued in the wild:**
- Hidden `.claude/` folder invisible by default on Windows (Explorer hides dotfiles)
- CRLF vs LF line endings break bash scripts inside skills
- OneDrive sync conflicts on user-profile-synced setups
- Antivirus quarantines the `.claude/` folder (especially corporate AV)
- Slow file I/O when skills live on `/mnt/c/` (9P protocol overhead)
- Recommendation in the wild: "store skills in WSL native FS for performance" — which *disconnects* you from the Windows side

**Why this is depth-layer info:** Nobody's sync tool solves this. The closest is skillshare's NTFS junction support, but junctions don't bridge the Windows↔WSL boundary — WSL sees the Linux filesystem, junctions are a Windows-side construct. The actual fix is bind-mounting or two-way file sync via `inotify`+`ReadDirectoryChangesW`, or a daemon that watches both sides — none of which exist in shipped form.

### Secrets leak at scale, and existing sync tools encourage it

The single most important Layer-3 finding:

> "GitGuardian's State of Secrets Sprawl 2026 report found that 24,008 unique secrets were discovered in MCP configuration files on public GitHub repositories. Of these, 2,117 were confirmed valid — live credentials capable of authenticating against real services."
> — [Token Security blog](https://www.token.security/blog/how-to-stop-exposing-secrets-on-your-mcp-configs)

> "AI-assisted commits had a 3.2% secret-leak rate, versus a 1.5% baseline across all public GitHub commits — roughly 2x the baseline."
> — [Snyk State of Secrets](https://snyk.io/articles/state-of-secrets/)

**Why it happens** ([dev.to: "Why Every MCP Setup Guide Is Teaching You to Store API Keys Wrong"](https://dev.to/the_seventeen/why-every-mcp-setup-guide-is-teaching-you-to-store-api-keys-wrong-4ghf)): the official quickstart docs show API keys inlined in JSON config examples. Developers copy the pattern. Sync tools then move that pattern to git. Boom.

**What existing sync tools do about it:** essentially nothing.
- vsync: "MCP configs with secrets should use environment variables" (i.e. user's problem)
- sync-agents-settings: same
- skillshare: built-in security audits *detect* secrets but don't manage them
- Claude Code Plugin marketplace: no first-class secrets contract

**What works elsewhere:**
- `get-mcp-keys` loads from `~/.mcp-keys/` outside git — single-machine only
- [macOS Keychain integration](https://kahunam.com/articles/automations-ai/securing-mcp-server-secrets-with-macos-keychain/) — single-OS only
- Chezmoi + [age](https://github.com/FiloSottile/age) or [sops](https://github.com/getsops/sops) — mature, cross-device, but requires per-user setup

**Implication for the user's tool:** a sealed bundle format with built-in secret references (e.g. `{{ secret "GITHUB_TOKEN" }}` with pluggable backends: env, keychain, age, sops, 1Password CLI) is a genuine and unfilled niche. This is probably the highest-leverage single feature.

### Cross-agent format conversion is mostly solved (the easy part)

The 4-axis matrix that any sync tool has to handle:

| Axis | JSON | JSONC | TOML |
|---|---|---|---|
| Config root key | `mcpServers` (Claude, Cursor) | `mcp` (opencode) | `mcp_servers` (Codex) |
| Env var syntax | `${VAR}` | `${env:VAR}` | `${VAR:-default}` literally expanded |
| Remote server shape | `type:"http"` with `url` | `type:"remote"` with `url` | `[mcp_servers.x] url=` |
| Comments | ❌ stripped | ✅ preserved | ✅ |
| Trailing commas | ❌ | ✅ | ✅ |

vsync and sync-agents-settings have already done this work. **This is not where the value is for a new tool.** Reuse or fork.

### The format/standard convergence story

There's a genuine race between three converging-but-distinct standards:

1. **AGENTS.md** (Linux Foundation, Agentic AI Foundation, Dec 2025 onward) — wins for **instructions** (CLAUDE.md / .cursorrules / GEMINI.md → one file). 30+ agents read it. Has *no required fields*. Won't extend to skills/MCPs structurally; it's just a markdown contract.

2. **MCP Registry** (live since Sept 2025, Linux Foundation since Dec 2025) — wins for **MCP server discovery**. ~2,000 servers. Acts "like DNS root servers — backbone not frontend." Doesn't solve local config sync but provides canonical metadata that a sync tool can consume.

3. **Claude Code Plugin format** (`.claude-plugin/plugin.json` + `marketplace.json`) — currently the most expressive *bundle* format (skills+MCPs+hooks+commands in one manifest). Official JSON schemas published at schemastore.org. **But it's Claude-only.** If OpenAI/Cursor/opencode adopt it, the format wins everything. If they don't, it stays a silo.

**Probability of cross-vendor plugin format convergence within 12 months: LOW.** Vendors have competitive incentive to keep formats sticky. AGENTS.md took 18+ months of OpenAI sponsorship to spread; plugins are a deeper commit (executable code, hooks, permissions). Bet on fragmentation persisting.

### Bidirectional sync is genuinely hard and nobody solves it

Every existing tool is one-way push. None handles the case:

- You install MCP server X in Claude Code on Monday
- You install MCP server Y in Cursor on Tuesday
- You want both on both, plus on your laptop

This is a 3-way merge problem and existing tools either (a) overwrite from a single source of truth, or (b) skip existing entries (sync-agents-settings does this — meaning re-runs don't update). The user's "smooth transition" framing implies bidirectional intent. **This is a hard problem with real value.**

Chezmoi solves it for dotfiles via git-merge semantics. Borrowing that model (treat configs as a git repo, render to per-agent destinations on `apply`) is the cleanest architectural pattern.

---

## Adjacent Opportunities

### A1. The "sealed bundle" format

**What it is:** A signed, secrets-aware bundle file (e.g. `skillpack.tar.zst`) that contains skills + MCPs + hooks + commands + a manifest specifying which agents it targets, with all secrets referenced by URI (`secret://keychain/X`, `secret://env/X`, `secret://age/key.txt#path`) instead of inlined.

**Why it's relevant:** The single biggest unfilled niche. Solves the team-sharing job *and* the secret-leak risk simultaneously. Composable with existing sync tools (you bring your own transport).

**Signal strength:** HIGH. 24k leaked secrets is a real number; no tool solves it; format design is the user's chance to lead.

**Next step to explore:** Look at [sops](https://github.com/getsops/sops) and [age](https://github.com/FiloSottile/age) integration patterns. Sketch a manifest schema, get feedback from maintainers of vsync / skillshare.

### A2. Cursor-style cloud Settings Sync, but cross-agent

**What it is:** A hosted sync service that does Cursor's "Settings Sync" (cross-device, encrypted, opt-in) but federated across Claude / Codex / opencode etc.

**Why it's relevant:** Solves cross-device sync without requiring users to set up dotfiles tooling. The reason people use Cursor's sync is the zero-config experience.

**Signal strength:** MEDIUM. Hosting is operationally expensive; trust model is hard (you're sitting in the middle of secrets). Possibly an enterprise product, not a solo-dev one.

**Next step to explore:** Survey what enterprise teams pay for in dotfiles space (most use git + cloud blob storage, not SaaS). Likely defer this.

### A3. Windows-WSL bidirectional filesystem watcher

**What it is:** Standalone daemon that watches `C:\Users\<u>\.claude\skills\` and `\\wsl$\<distro>\home\<u>\.claude\skills\`, propagating changes both ways with conflict resolution. Handles CRLF↔LF auto-translation, executable bit translation, ignores OS-junk files.

**Why it's relevant:** Surgically solves the user's exact stated pain. Composable with anything else.

**Signal strength:** MEDIUM-HIGH for the user personally; MEDIUM for the wider audience. Smaller TAM but very clearly differentiated.

**Next step to explore:** [Unison file synchronizer](https://github.com/bcpierce00/unison) (mature, cross-platform 2-way sync, but heavyweight). Watch APIs: `ReadDirectoryChangesW` (Windows) and `inotify` (WSL). Likely a separate sibling tool, not part of the main exporter.

### A4. AGENTS.md skill extension

**What it is:** Propose to the Agentic AI Foundation that AGENTS.md grows a `skills/` and `mcp/` section (frontmatter or fenced sections). Become the OSS champion who lands the cross-agent skill standard.

**Why it's relevant:** Bets on the meta game. If the standard happens, your tool is the reference implementation. If it doesn't, you've still built a useful tool.

**Signal strength:** MEDIUM. Spec work is slow and political. AGENTS.md took OpenAI sponsorship to spread.

**Next step to explore:** Read [agents.md spec discussion](https://www.morphllm.com/agents-md-guide), look at PRs in the standard's governance repo.

### A5. Cross-agent permission/allowlist sync

**What it is:** Claude Code, Cursor, Codex all have allow/deny rules for tool use. They use different formats. Sync them.

**Why it's relevant:** Adjacent to MCP sync, growing in importance as agent autonomy grows. Currently nobody does this in a principled way.

**Signal strength:** MEDIUM. Becomes high-importance as enterprise adoption grows (audit/compliance angle).

**Next step to explore:** [claude-code-permission-manager skill](https://mcpmarket.com/tools/skills/claude-code-permission-manager) as a starting point.

### A6. Skill + MCP signed-provenance registry

**What it is:** Like npm's [provenance attestations](https://docs.npmjs.com/generating-provenance-statements), but for skills/MCPs. When you publish a skill bundle, sign it; when you install, verify it came from the claimed source. Counters the supply-chain risk of skills running shell commands.

**Why it's relevant:** As AGENTS.md skill counts hit 1500+ (antigravity-awesome-skills) and hooks execute arbitrary shell commands, supply chain attack surface grows.

**Signal strength:** MEDIUM today, HIGH within 12 months as incidents happen.

**Next step to explore:** [sigstore](https://www.sigstore.dev/) as a primitive. Out of scope for MVP but worth keeping in mind.

---

## Competing Perspectives & Counterarguments

A research conclusion needs to survive its strongest attack. Here are the five strongest attacks:

### Attack 1: "Anthropic will absorb this into Claude Code Plugins within 6 months."

**The strongest case:** The Claude Code plugin marketplace already covers the team-sharing job. JSON schemas are open. If Anthropic adds a "compatibility layer" that exports plugins to Cursor/Codex format, the third-party sync tools become wrappers.

**Counter:** AGENTS.md took **18+ months** of OpenAI sponsorship and Linux Foundation governance to spread. Anthropic has no incentive to ship a tool that helps users move away from Claude Code; if anything, the plugin marketplace is a lock-in vector. The OS-boundary (Windows ↔ WSL) is *not Anthropic's problem to solve at all* — it's an OS-level issue. Even if Anthropic ships cross-agent plugins tomorrow, the WSL split persists.

**Verdict:** Risk is real for the team-sharing scenario only. Mitigated by focusing differentiation on the OS-boundary, secrets, and bidirectional sync — none of which Anthropic has signalled interest in.

### Attack 2: "Skillshare has 2.2k stars and momentum. Why not just contribute there?"

**The strongest case:** Skillshare is established, actively maintained (v0.20.14 in June 2026), Go binary, cross-platform. A PR adding MCP support and a sealed-bundle format gets you 100% of the value with 10% of the effort and 1000x the audience.

**Counter:** Skillshare's architectural choices (Go binary, no MCP, no hooks, git-only distribution) reflect deliberate scope. Adding MCP + hooks + a new bundle format would be a major architectural change — probably easier to build separately and integrate. **But this is the weakest counter** — contributing first is probably the right move tactically.

**Verdict:** Genuine. Recommendation in [Recommendations] below incorporates this: prototype as a *complementary tool* that hooks into skillshare/vsync, not as a head-on competitor.

### Attack 3: "Windows-WSL is a niche pain — most devs are on Mac or single-OS."

**The strongest case:** Anthropic's open issue has visibility but no critical mass. macOS dominates the dev influencer demographic. The pain might be loud but small.

**Counter:** Windows + WSL is enormous in enterprise. WSL had 5M+ MAU in 2023 and grew. Corporate developers on Windows hardware running Linux toolchains is a major slice of the actual paid Claude Code audience (vs. the influencer slice). Plus the user is hitting it personally — sample-of-one matters when the user is the builder.

**Verdict:** Pain is smaller than secrets, but real. Worth a sibling tool, not the main pitch.

### Attack 4: "Configuration sync is solved (chezmoi/yadm). Anything more specific is overengineering."

**The strongest case:** Multiple blog posts show developers using chezmoi *successfully* for AI agent configs across machines and tools. Why reinvent it?

**Counter:** Chezmoi requires hand-rolled templates per agent, manual age-encryption setup, no built-in knowledge of any agent's schema. Half-right: the right product might be a chezmoi *plugin/module* that ships agent-aware templates and schema validators. That's smaller scope, higher leverage, lower competition.

**Verdict:** Genuine. **A chezmoi-agent-config module is a viable smaller-scope alternative MVP** — flag it as Decision Point B below.

### Attack 5: "By the time you ship, format wars resolve and your tool is obsolete."

**The strongest case:** AGENTS.md surprised everyone with adoption speed. If a "skill manifest" standard emerges in the next 6 months, transformer tools become obsolete.

**Counter:** Half-right but planable. Build *adapters* not *transformers*. If a standard lands, adapters drop out. Position the tool around the gaps standards won't fill (OS-boundary, secrets, bidirectional sync) — those persist regardless of format convergence.

**Verdict:** Real but manageable. Architectural posture mitigates.

---

## Decision Points

The research surfaces four reasonably distinct scopes. The user should pick before writing the PRD.

### Option A: Full "skill exporter" — everything

- **Scope:** MCPs + skills + hooks + commands + subagents + permissions, across all 4 boundaries, with sealed bundles and secrets.
- **Leads to:** Direct competitor to vsync + skillshare combined, with extras. Most flexible product.
- **Sacrifices:** 12+ months of work for solo developer. Will get outflanked by Anthropic or by the existing tools shipping incrementally.
- **When this wins:** If you want a SaaS/business out of it later, or are willing to commit long-term.

### Option B: Chezmoi-agent module — leverage mature dotfiles

- **Scope:** Ship a chezmoi-aware module that knows agent schemas, validates configs, handles secrets via chezmoi's existing `age`/`sops` integration. User installs chezmoi; gets agent-aware sync.
- **Leads to:** Working cross-device + team-sharing tool in 2-4 weeks. Reuses chezmoi's transport/secrets primitives. Minimal scope.
- **Sacrifices:** Anyone not already on chezmoi has to adopt it. Limits audience.
- **When this wins:** If you want fast time-to-value and small surface area to maintain.

### Option C: Sealed bundle format only — orthogonal niche

- **Scope:** Define a portable, signed, secrets-aware bundle format (`skillpack.json` + tarball). Ship a reference CLI to pack/unpack/install. Integrate as plugin/extension to existing tools (skillshare, vsync, Claude Code Plugin marketplace).
- **Leads to:** Tightly scoped 4-6 week build. Becomes the *missing layer*. Composable.
- **Sacrifices:** Doesn't solve cross-device or Windows-WSL by itself. Users need other tools.
- **When this wins:** If you want maximum leverage per LOC, want to be the "format guy" not the "tool guy."

### Option D: Windows-WSL bidirectional sync daemon — surgical

- **Scope:** Single-purpose daemon that watches both filesystems and propagates skills/configs both ways. Handles CRLF, exec bits, ignores junk files. CLI for one-shot sync.
- **Leads to:** Solves the user's exact stated pain in 2-3 weeks. Standalone utility, no competition.
- **Sacrifices:** Niche audience. Doesn't help cross-agent or team-sharing.
- **When this wins:** If the user wants to scratch the personal itch fast and ship something visible.

**Most defensible combination for a real product:** **C + D**, with B as a fallback if scope creep happens. C is the differentiator; D is the personal pain killer; both can ship independently and compose later.

---

## Horizon: Worth Knowing About

Five things the user didn't ask about but should hold in peripheral vision:

1. **[claw-orchestrator](https://github.com/Enderfga/claw-orchestrator)** — runs Claude/Codex/Gemini/Cursor as one unified runtime. If multi-agent orchestration is where the world goes, *running* multiple agents matters more than *syncing configs*. Watch trajectory.

2. **[Smithery skill marketplace](https://smithery.ai/)** and **[agentskills.io](https://agentskills.io/)** — emerging curation layers. If a marketplace standard solidifies, all sync tools become "install from marketplace" wrappers.

3. **[npm provenance attestations](https://docs.npmjs.com/generating-provenance-statements)** — supply chain signing model. As skills run arbitrary shell commands in hooks, this becomes load-bearing within 12 months.

4. **[`sigstore` / `cosign`](https://www.sigstore.dev/)** — keyless code signing infrastructure. Direct primitive for sealed-bundle signing in Option C.

5. **GitHub MCP Registry integration** ([blog post](https://github.blog/ai-and-ml/generative-ai/how-to-find-install-and-manage-mcp-servers-with-the-github-mcp-registry/)) — GitHub may end up as the discovery layer, in which case the user's tool should consume GitHub's APIs rather than build its own catalog.

---

## Recommendations: Concrete Scope for the User's Tool

After framework synthesis and red-teaming, here is the recommended scope. The PRD that follows should crystallize these.

### Recommended MVP (4-6 weeks part-time):

**Tool name (placeholder):** `agentpack` or similar.

**Core deliverable:** A CLI + bundle format that:

1. **Defines `agentpack.yaml`** — a manifest that declares which artifacts (skills, MCPs, hooks, commands, agents) the bundle ships and which target agents it supports.
2. **Bundles to a tarball** with deterministic content hashing, optional signing via `cosign` or minisign.
3. **References secrets by URI**, never inlining them: `env://`, `keychain://`, `age://`, `sops://`, `1p://op://vault/...`. The bundle alone is *safe to share publicly*.
4. **Renders/installs to target agent's config layout** — leverages vsync's transformation work (potentially as a dependency).
5. **Ships an `agentpack export ~/.claude` command** that captures the user's current Claude Code setup into a bundle in one shot — that is the "skill exporter" for the user's headline use case.
6. **Ships an `agentpack install <bundle>` command** that targets a list of agents found on the system.

### Phase 2 (3-4 additional weeks):

- **Bidirectional sync via git remote** — store the manifest in a git repo, `agentpack pull` reconciles diffs.
- **Windows ↔ WSL bridge utility** as a sibling tool (`agentpack wsl-sync`).
- **Chezmoi adapter** — emit chezmoi templates from an agentpack bundle for users already on that stack.

### Explicitly defer (NOT in MVP):

- Plugin marketplace (use git URLs)
- GUI / desktop app
- Live watch mode
- Custom registry / SaaS hosting
- Conflict-resolution UI for bidirectional sync

### Why this scope wins:

- **No direct competitor** ships sealed, secrets-aware bundles. That's the moat.
- **Composes with the ecosystem** instead of competing — can pack into Claude Code Plugin marketplace format, can be a Cursor/Codex install target, can use vsync internals for transforms.
- **Survives format convergence** — if AGENTS.md grows skill sections, agentpack becomes the binary that emits them.
- **Solves the user's personal pain** (export current Claude setup → install in WSL) within the MVP scope.

### Open scope questions the PRD should resolve:

1. Language/runtime: Go (skillshare's choice, single binary), Node (vsync's choice, npm-installable), or Rust? Tradeoff: Go gets you single-binary distribution; Node gets you `npx` install but Node dependency.
2. Signing scheme: `cosign` keyless (modern but Sigstore-dependent) vs. `minisign` (simpler, key-only)?
3. Secret URI scheme: ship with which backends in MVP? (Recommend `env://` and `keychain://` minimum.)
4. Manifest format: YAML (human-friendly), TOML (matches Codex), or JSON (matches Claude/Cursor)? Recommend YAML.
5. Bidirectional reconciliation strategy: last-writer-wins, three-way merge via git, or explicit lockfile-driven?

---

## Sources & Confidence

### Primary sources (HIGH confidence claims):

- **Direct competitor tool READMEs and docs** ([vsync](https://github.com/nicepkg/vsync), [sync-agents-settings](https://github.com/Leoyang183/sync-agents-settings), [skillshare](https://github.com/runkids/skillshare), [antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills)) — covers scope, status, stars, format handling
- **Anthropic's Claude Code docs** ([Plugin reference](https://code.claude.com/docs/en/plugins-reference), [advanced setup](https://code.claude.com/docs/en/setup), [WSL Issue #49933](https://github.com/anthropics/claude-code/issues/49933))
- **OpenAI Codex docs** ([Config basics](https://developers.openai.com/codex/local-config/), [MCP](https://developers.openai.com/codex/mcp))
- **opencode docs** ([Config](https://opencode.ai/docs/config/), [MCP servers](https://opencode.ai/docs/mcp-servers/))
- **MCP Registry blog** ([Sept 2025 announcement](https://blog.modelcontextprotocol.io/posts/2025-09-08-mcp-registry-preview/))
- **Anthropic donation to Linux Foundation** ([Dec 2025](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation))

### Secondary sources (MEDIUM confidence claims):

- **GitGuardian / Snyk State of Secrets 2026** ([Token Security analysis](https://www.token.security/blog/how-to-stop-exposing-secrets-on-your-mcp-configs), [Snyk article](https://snyk.io/articles/state-of-secrets/), [dev.to: "29 Million Secrets Leaked"](https://dev.to/mistaike_ai/29-million-secrets-leaked-on-github-last-year-ai-coding-tools-made-it-worse-2a42)) — secrets-leak numbers
- **AGENTS.md guides** ([Morph](https://www.morphllm.com/agents-md-guide), [Vibecoding](https://vibecoding.app/blog/agents-md-guide), [Crozat](https://benjamincrozat.com/agents-md)) — adoption claims
- **Windows/WSL paths reference** ([Agensi guide](https://www.agensi.io/learn/claude-code-skills-windows-complete-guide), [Inventive HQ Codex location](https://inventivehq.com/knowledge-base/openai/where-configuration-files-are-stored))

### Tertiary sources (LOWER confidence — single source or marketing material):

- Star counts from search snippets (snapshotted at query time, may have moved)
- "60+ tools" claim from skillshare README — not enumerated
- Claude Code Plugin marketplace exact size — multiple conflicting figures (425 plugins / 2810 skills in one source; "5 bundled skills + official anthropics/skills marketplace" in another)

### Sources I could not access:

- **Reddit r/ClaudeAI threads** — fetch was blocked. The pain-signal claims for Reddit are inferred from blog post mentions, not direct thread reading. **Confidence on grassroots Reddit sentiment is LOW.** If the user has Reddit access, a manual scan of r/ClaudeAI, r/cursor, r/ChatGPTCoding for keywords "sync" / "WSL" / "share skills" would harden this.
- **HackerNews discussions** — not directly searched; would strengthen the "this is hot" claim.

### Where this brief is thinnest:

- **Adoption velocity of the direct competitors** — star counts are a weak proxy. Number of weekly downloads / actual installs would be more telling but isn't visible.
- **Cursor's internal cloud Settings Sync feature** — referenced but not deeply analyzed. Could compete on cross-device job for users already on Cursor.
- **Enterprise sync solutions** — there may be quiet B2B tools (e.g. for compliance) that don't surface in OSS searches.

### Overall confidence rating: **HIGH on the landscape and gap analysis, MEDIUM on the strategic recommendations.**

The recommendations are well-supported by the competitive map but depend on judgment calls (which scope wins, whether platforms eat the niche). The user should treat the four "Decision Points" (A/B/C/D) as the genuine fork and not skip the choice.

---

## Appendix: Mapping back to the four user-stated scenarios

| Scenario | State of the art | Recommended approach |
|---|---|---|
| **OS boundary (Win↔WSL)** | No tool natively bridges; gotchas catalogued but unsolved | **Phase 2 / Option D** — sibling daemon, surgical fix |
| **Agent boundary** | Most crowded space (vsync, sync-agents-settings, skillshare) | **Reuse** — depend on or fork vsync's transformation engine |
| **Cross-device** | Mostly solved by chezmoi-pattern but with manual setup | **Phase 2 / Option B fallback** — git-based via agentpack manifest |
| **Team sharing** | Claude Code Plugin marketplace covers it within Claude; nothing cross-agent | **MVP / Option C core** — sealed bundle format is the differentiator |

End of brief.
