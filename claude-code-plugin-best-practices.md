# Claude Code Plugin Best Practices

**Last Updated:** 2026-05-25
**Sources:** Hard-won lessons from `dbt-pipeline-toolkit` and `fabric-dataflow-migration-toolkit` (two production Claude Code plugins). All section cross-references map back to `plugin_learnings.md` findings in the source repos. External citations (changelog notes, blog posts, community videos) are embedded inline in Sections 15–18.

> **Companion documents:**
> - `how-to-create-plugins.md` — step-by-step build/release guide
> - `how-to-create-agents.md` — step-by-step agent guide
> - `how-to-create-skills.md` — step-by-step skill guide
> - `how-to-create-hooks.md` — step-by-step hook guide
> - `claude-code-agent-best-practices.md` — agent-level patterns (includes plugin section)
> - `claude-code-skill-best-practices.md` — skill-level patterns (includes plugin section)
> - `claude-code-hooks-best-practices.md` — hook patterns (includes plugin section)

## Table of Contents

1. The Core Insight: Dev vs Installed Behavior Diverges
2. Plugin Architecture Principles
3. Namespace Rules (and Why They Bite You Silently)
4. The Permission Model & Background Subagents
5. The Atomic-Bash Principle
6. The Security Boundary: Plugin Manifest vs Agent Frontmatter
7. State, Reference Materials, and Where They Live
8. Orchestrator Topology
9. User Interaction Budget
10. Testing Strategy (Fresh Install Is the Only Truth)
11. Observability & Failure Surfacing
12. Release Engineering for Plugins
13. Top 16 Anti-Patterns to Avoid
14. Themes
15. Source Validation: Official Corroborations & Open Questions
16. Verbatim Quotes Library (slide-ready)
17. Community Video References
18. Anthropic Engineering Posts: The Canon

---

## 1. The Core Insight: Dev vs Installed Behavior Diverges

This is the meta-lesson behind everything else in this document. **Dev mode is not a representative test environment.** When you point Claude Code at your repo with `--plugin-dir ./`, or drop your agents directly into `~/.claude/agents/` to iterate quickly, you are running with a different namespace, a different permission surface, a different filesystem layout, and a different interactive channel than the user who installs your plugin from a marketplace will get.

Code that looks correct, runs cleanly, and produces the expected output in dev mode can fail silently on a fresh install for at least six independent reasons — none of which surface as an error message.

| Aspect | Dev (`--plugin-dir ./` / standalone) | Installed (via marketplace) |
|---|---|---|
| Agent name | `business-analyst` | `my-plugin:business-analyst:business-analyst` (3-part) |
| `permissionMode` in agent frontmatter | Respected | Silently stripped (DBT F2) |
| `hooks` / `mcpServers` on an agent | Respected | Silently stripped (DBT F2) |
| Script paths via `$HOME/.claude/skills/<name>/...` | Work (you have a standalone copy) | Point nowhere (DBT F7) |
| Bare-name agent references | Work | Resolve to nothing (DBT F1) |
| Background subagent Bash | Works (you can answer prompts) | Stalls (no interactive channel) (DBT F3, F9) |
| `userConfig` env vars in subprocesses | Bare names available if you exported them | Prefixed `CLAUDE_PLUGIN_OPTION_<KEY>` only (DBT F5) |
| Plugin-cache file reads from subagents | Work | Blocked (Fabric N14) |

Every row in this table is a class of bug whose root cause is "dev mode hid the breakage." Most of the rest of this document is a guided tour through each row — what the breakage looks like, why it happens, and what the fix is. But the single most important best practice is upstream of all of those fixes:

> **Test on a fresh install before declaring "done."** A clean machine, or at minimum a wipe of `~/.claude/plugins/cache/` followed by `/plugin install ...` from your marketplace, is the only environment that reproduces what your users see.

Dev shortcuts are useful for iteration speed. They are not a substitute for fresh-install verification, and they actively hide the bugs that hurt the most. If you can only afford to adopt one practice from this document, adopt this one.

---

## 2. Plugin Architecture Principles

The DBT and Fabric plugins both converged on the same five architectural principles after several painful refactors. None of them are obvious from the docs.

### Single responsibility per agent, orchestrator + specialists

Each agent owns exactly one role: requirements gathering, profiling, staging, dimensions, facts, tests, validation, deployment, etc. The orchestrator is the only agent that holds global pipeline state — and it never does substantive work itself. It plans, fans out to specialists, verifies their output, and either advances to the next stage or halts.

This pattern matches the agent hierarchy constraint (Section 8) and the user-interaction budget (Section 9). Trying to bake multi-stage logic into a single mega-agent always regresses to a less reliable version of the orchestrator pattern, because you can't observe an agent's intermediate state from the outside the way you can observe a specialist's envelope from the orchestrator.

Anthropic's own framing of this pattern (Building agents with the Claude Agent SDK, 2025-09-29):

> "Subagents are useful for two main reasons. First, they enable parallelization… Second, they help manage context." — https://claude.com/blog/building-agents-with-the-claude-agent-sdk

### Plugin = security perimeter

Anything privileged — hooks, MCP servers, Bash auto-approval, lifecycle event handlers — must be declared in `plugin.json` rather than agent frontmatter. Claude Code's plugin loader treats the manifest as the audit surface: users can inspect it before enabling the plugin, and reviewers can grep it for sensitive declarations. Agent frontmatter is unaudited spawn-time metadata, so the loader strips elevated fields on plugin install. Section 6 covers the asymmetry in detail; the architectural point here is that the perimeter is at the plugin boundary, not at the agent boundary.

The official plugins reference makes the policy explicit:

> "For security reasons, `hooks`, `mcpServers`, and `permissionMode` are not supported for plugin-shipped agents." — https://code.claude.com/docs/en/plugins-reference

### Manifest as contract

`plugin.json` is the public contract for what the plugin does, what it asks for, and what it ships. Push declarations there even when the same effect could be achieved at the agent level. The audit surface goes up; the surprise factor goes down; the maintenance burden moves from "where is this configured?" guesswork to one source of truth.

### Two-repo model for distribution

The plugin repo holds the agents, skills, hooks, MCP servers, manifest, and tests. The marketplace repo holds an index pointing at one or more plugin repos. Users install via the marketplace:

```
/plugin marketplace add OrgName/Marketplace-Repo
/plugin install my-plugin@MarketplaceName
```

Both the DBT and Fabric plugins follow this pattern (each lives in its own GitHub repo; both are indexed from a shared `AI-plugins` marketplace repo). Mixing the two — putting `marketplace.json` into the plugin repo — breaks `/plugin install` behavior and confuses the loader. Keep them separate.

Anthropic's launch post confirms the marketplace bar is intentionally low:

> "To host a marketplace, all you need is a git repository, GitHub repository, or URL with a properly formatted `.claude-plugin/marketplace.json` file." (2025-10-09 — Customize Claude Code with plugins — https://claude.com/blog/claude-code-plugins)

### Bundle the dependencies you can

A plugin that needs the user to `npm install ...`, `pip install ...`, or hand-edit `settings.json` before it works has a broken onboarding. Whatever can be bundled, bundle:

- **MCP servers:** ship pre-built `dist/` artifacts inside the plugin (the DBT plugin's `servers/dist/minimal-mcp-server.js` is shipped compiled).
- **Python dependencies:** pin them inside skill scripts and detect missing imports loudly, with remediation instructions in the error.
- **Reference materials:** copy them into the project at scaffold time so background subagents can read them (Section 7, Fabric N14).
- **Sample inputs:** bundle `examples/` so users can run `--sample --dry-run` end-to-end without their own data (Fabric N6).

Detect missing prerequisites in a SessionStart hook (Fabric ships `hooks/session-start-config-check.py`) and surface the remediation before the user hits the failure.

---

## 3. Namespace Rules (and Why They Bite You Silently)

Plugin namespacing has three rules, none of which are fully documented and one of which contradicts the docs example. Get them wrong and every cross-reference inside your plugin silently resolves to nothing.

### Rule 1: Skills are flat (2 parts)

```
my-plugin:data-profiler
```

A skill lives at `skills/<name>/SKILL.md`. The directory IS the skill identifier — there's no intermediate level. The registered name is always 2-part. (DBT F8.)

Corroborated by the plugins reference:

> "Plugin skills use a `plugin-name:skill-name` namespace, so they cannot conflict with other levels." — https://code.claude.com/docs/en/plugins-reference

And by v2.1.142 (2026-05-14):

> "Plugins with root-level `SKILL.md` and no `skills/` subdirectory now surfaced as skill." — flat-layout skill plugins are explicitly supported.

### Rule 2: Subdirectory agents are 3-part

```
my-plugin:business-analyst:business-analyst
```

When agents live at `agents/<name>/agent.md` (the layout the DBT and Fabric plugins both use), Claude Code treats the subdirectory as a separate namespace segment above the frontmatter `name` field. The registered name is 3-part. (DBT F1.)

**This 3-part form is not announced in any release note.** It is consistent with v2.1.145's `/plugin` Discover panes listing "commands, agents, skills, hooks, and MCP/LSP servers" per plugin, but never documented explicitly. Verify on a fresh install via the `/agents` picker.

### Rule 3: Flat agents are 2-part

```
my-plugin:security-reviewer
```

When agents live at `agents/<name>.md` (the layout shown in the official docs example), there's no intermediate directory, so the registered name is 2-part. **This is the case the docs example shows — and the case that misleads everyone using subdirectory layouts.**

### The directory name must equal the frontmatter `name`

If `agents/business-analyst/agent.md` has `name: ba-bot` in its frontmatter, the registered name silently follows the **directory** (`my-plugin:business-analyst:ba-bot` — not `my-plugin:ba-bot:ba-bot`). Every in-repo reference to `business-analyst` resolves correctly; references using `ba-bot` resolve to nothing. The convention is to keep them identical so the duplicated middle and last segments visually signal "this is the registered 3-part name."

### Every cross-reference must use the registered name

Once you know the 3-part form, every reference in your plugin must use it:

```yaml
# orchestrator agent.md frontmatter
tools: >-
  Agent(my-plugin:business-analyst:business-analyst,
        my-plugin:data-explorer:data-explorer,
        my-plugin:dbt-staging-builder:dbt-staging-builder),
  Read, Write, Bash
```

```python
# Task spawn
Task(
  subagent_type: "my-plugin:business-analyst:business-analyst",
  prompt: "...",
  run_in_background: true,
  mode: "acceptEdits"
)
```

```bash
# CLI invocation
claude --agent my-plugin:orchestrator:orchestrator "Build a pipeline"
```

v2.1.140 made this more forgiving:

> "Improved Agent tool `subagent_type` matching (case/separator insensitive)" — case/separator insensitive lookup, but the underlying 3-part form is still required.

### `skills:` frontmatter uses the 2-part form (different from agent references)

This is the trap. The same agent that references a sibling agent via the 3-part form references its preloaded skills via the **2-part** form:

```yaml
# agent.md frontmatter
skills: my-plugin:dbt-runner, my-plugin:data-profiler, my-plugin:sql-server-reader
```

Bare names (`dbt-runner`) silently resolve to nothing — the skill preload returns an empty list and the specialist starts without its declared skill context. (DBT F8.)

### Verify on a fresh install via the `/agents` and `/skills` pickers

The pickers are the source of truth for registered names. Don't trust the docs (the 2-part example is only correct for the flat-file layout). Don't trust dev mode (it uses bare names). Install on a clean machine, type `/agents` and `/skills`, and confirm the names match what your orchestrator references. Both plugins now ship pre-shipment audits (`tests/preshipment_audit.py` with a `gate_namespace` gate) that grep every cross-reference and fail if any uses a bare name.

---

## 4. The Permission Model & Background Subagents

Background subagents (`run_in_background: true`) have no interactive channel. Any tool call that triggers a permission prompt has nowhere to send it, and the call stalls indefinitely. The orchestrator polls, sees no progress, eventually times out, and reports "no models built" with no usable error.

### `acceptEdits` doesn't cover what its name implies

```
"acceptEdits: Automatically accepts file edits and common filesystem commands
 (mkdir, touch, mv, cp, etc.) for paths in the working directory or
 additionalDirectories"
```

`acceptEdits` covers `Write`, `Edit`, and a narrow set of filesystem Bash atoms (`mkdir`, `touch`, `mv`, `cp`, `rm`). It does **not** cover `python ...`, `dbt run`, `fab job`, `pwsh -File ...`, or any other CLI invocation. Every Python script your plugin needs to run in a background subagent is blocked by default — including the data profiler, the SQL loader, the dbt runner, and every export/deploy script. (DBT F9.)

### `bypassPermissions` is too broad

Setting `mode: "bypassPermissions"` at the spawn site unlocks every Bash command for the spawned subagent — including ones that have nothing to do with your plugin. The user's global CLAUDE.md explicitly discourages this for background contexts, and it's the wrong answer for shipping a plugin: the security perimeter is supposed to be visible in `plugin.json`, not implicit at every spawn site.

Anthropic explicitly acknowledges the danger:

> "`--dangerously-skip-permissions` bypasses protected paths." (Changelog v2.1.129 and v2.1.126)

### Solution: plugin-level PreToolUse Bash allowlist hook

The Claude Code permission docs explicitly support this:

> "PreToolUse hooks run before the permission prompt. The hook output can deny the tool call, force a prompt, or skip the prompt to let the call proceed." — https://code.claude.com/docs/en/hooks

A PreToolUse hook that returns `permissionDecision: "allow"` bypasses the permission prompt entirely — which is exactly what background subagents need. Both shipped plugins use this pattern:

```json
{
  "matcher": "Bash",
  "hooks": [
    {
      "type": "command",
      "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/approve-plugin-bash.py",
      "statusMessage": "Checking plugin Bash allowlist..."
    }
  ]
}
```

The hook script reads the PreToolUse JSON from stdin, matches the Bash command against a narrow allowlist of plugin-internal patterns (`python "${CLAUDE_PLUGIN_ROOT}/skills/*/scripts/*.py" ...`, `dbt run|test|build|...`, `fab job|api|...`, etc.), and emits an allow decision for matches. Anything not on the allowlist falls through to the normal permission flow. The user audits the allowlist by grepping one file. (DBT F9; Fabric inherits.)

IndyDevDan's canonical hooks framing matches this exactly:

> "Hooks are the only place to enforce policy that a sub-agent cannot bypass." — Video: "I'm HOOKED on Claude Code Hooks: Advanced Agentic Coding" — https://www.youtube.com/watch?v=J5B9UGTuNoM

### Pass `mode` at the call site, not in agent frontmatter

```python
Task(
  subagent_type: "my-plugin:dbt-dimension-builder:dbt-dimension-builder",
  prompt: "...",
  run_in_background: true,
  mode: "acceptEdits"   # <- mandatory for background subagents
)
```

Agent frontmatter `permissionMode: acceptEdits` is silently stripped on install (DBT F2). The supported equivalent is `mode:` at the Task call site, which the parent agent owns. Both shipped plugins removed `permissionMode:` from every specialist frontmatter and pushed it to the orchestrator's spawn calls instead.

v2.1.143 added a related escape hatch:

> "Added `worktree.bgIsolation: 'none'` setting for background sessions to edit working copy directly without `EnterWorktree`." — Changelog v2.1.143

### `AskUserQuestion` is not available in any subagent

Documented hard restriction in the Agent SDK user-input docs:

> "Subagents: `AskUserQuestion` is not currently available in subagents spawned via the Agent tool."

This is independent of foreground vs background — listing `AskUserQuestion` in a subagent's `tools:` frontmatter has no effect. The parent agent is the only place `AskUserQuestion` works. Section 9 covers the three-step parent-owned pattern that replaces "specialist asks the user something." (Fabric N15.)

Community-confirmed in Issue #34592:

> "`AskUserQuestion` is completely absent from sub-agent environments — not in direct tools, deferred tools, or `ToolSearch` results." — https://github.com/anthropics/claude-code/issues/34592

---

## 5. The Atomic-Bash Principle

Every Bash command — generated by Claude in an agent body, generated by Claude during a session, or run by a hook script — must be a single atomic operation. No `&&`, no `||`, no `;`, no `|`, no `|&`, no background `&`, no subshells `(...)`, no `$(...)`, no backticks, no heredocs, no non-essential redirects (`2>/dev/null`, `>/dev/null`).

This includes `cd <path> && <command>` — pass an explicit path to the binary instead (`git -C <path> ...`, `python --cwd ...` if the binary supports it, or just include the absolute path in the command's own arguments).

### Why this matters

Claude Code's permission layer matches rules per-subcommand. A compound expression forces the permission engine to split the command, evaluate each part, and decide whether the combination is permitted. The splitter has corner cases (quoted strings containing operators, escaped operators, subshell scoping); the corner cases produce permission prompts; permission prompts in background subagents stall silently. (DBT F9, Round 2.)

Atomic commands also play nicely with `acceptEdits`. The mode auto-approves atomic `mkdir`, `mv`, `cp`, `rm`, `touch` calls without involving any hook. Wrapping the same operations in a compound (`mkdir -p foo && touch foo/bar`) loses the auto-approval even though every piece individually qualified.

### The translation patterns

| If you want to | Don't write | Write instead |
|---|---|---|
| Run A then B | `A && B` | Two separate Bash tool calls in sequence |
| Run B if A fails | `A \|\| B` | Issue A, read the exit code in LLM text, conditionally issue B as a second call |
| Use A's output as B's input | `A \| B` | Issue A, parse the output in LLM text, issue B with extracted arguments |
| Many related steps | a compound shell pipeline | Write them into a Python or PowerShell script, call the script as one atomic invocation |
| Detect mode before next step | `ls foo && echo "YES" \|\| echo "NO"` | Run `ls foo`, read the exit code in LLM text, branch in the next call |
| Init or do-nothing repo | `git rev-parse --git-dir \|\| (git init && git add -A && git commit ...)` | Four atomic calls: `git rev-parse`, then on failure `git init`, `git add -A`, `git commit` |
| Count files | `ls *.csv \| wc -l` | `find "<dir>" -name "*.csv" -type f`, count lines in LLM text |

### Cost vs benefit

Each Bash tool call carries ~50–80 tokens of protocol overhead. Refactoring a compound into 3–4 atomic calls adds roughly 150–250 tokens per refactored step. Across a typical pipeline build session (100K–500K tokens, ~10–15 previously compound operations), the total cost is **~1–3% more tokens per session** — for a substantial reliability gain. Background subagents go from "stalls on first compound" to "works consistently."

The persistent cost is the ~400 tokens the rule itself adds to the global `~/.claude/CLAUDE.md` (which loads into every session). The DBT plugin's first draft of the rule was ~1,100 tokens; it was tightened to ~400 to keep the ambient overhead small.

### Belt and suspenders

The atomic-Bash rule belongs in every audience's file at once:

- The orchestrator's own `agent.md` body (so it's loaded at the orchestrator's session start).
- Every specialist's `agent.md` body (so it's loaded at every specialist's spawn).
- The project `CLAUDE.md` template the plugin scaffolds (so post-build sessions inherit it).
- The user's global `~/.claude/CLAUDE.md` (so every Claude Code session anywhere has it).
- The plugin's pre-shipment audit (`gate_atomic_bash` greps every agent body and SKILL.md for compound expressions and fails on hit).

Yes, that's ~5 places. Each audience reads a different file. Skip any one and that audience drifts. (DBT F9 Round 3.)

**Note:** No public video or blog post in our research surveyed (~35 videos, ~22 Anthropic blog posts) discusses the atomic-Bash rule explicitly. This is the strongest "new contribution" candidate for any conference talk derived from this doc — a community blind spot.

---

## 6. The Security Boundary: Plugin Manifest vs Agent Frontmatter

Claude Code enforces an asymmetric trust model between the plugin manifest and agent frontmatter. The asymmetry is deliberate, and understanding it is what unlocks the rest of the plugin capability surface.

| Field | In `plugin.json` | In agent.md frontmatter |
|---|---|---|
| `hooks` | Honored — fires on registered events | Stripped at load time (DBT F2) |
| `mcpServers` | Honored — server starts on plugin enable | Stripped at load time (DBT F2) |
| `permissionMode` | N/A (not a manifest field) | Stripped at load time (DBT F2) |

### Why the asymmetry exists

A plugin-level hook or MCP server is declared in `plugin.json`. Users can inspect that file before enabling the plugin — it's a fixed file in the source repo, greppable, visible in the marketplace listing, surfaced in `/plugin` inspection commands. The manifest is the plugin's public contract.

Agent frontmatter, by contrast, is spawn-time metadata that the user never explicitly opts into. Letting each agent attach its own hooks at spawn time would move privilege-granting into an unaudited surface. Letting an agent silently upgrade itself to `permissionMode: bypassPermissions` would be privilege escalation without user awareness. So Claude Code strips those fields on install — not as a bug, as the policy.

Plugins-reference explicit policy quote:

> "For security reasons, `hooks`, `mcpServers`, and `permissionMode` are not supported for plugin-shipped agents." — https://code.claude.com/docs/en/plugins-reference

v2.1.147 patched a related silent-drop bug that hid an asymmetric drop in *manifest-loaded* frontmatter:

> "Fixed plugin agents declaring multiple `Agent(...)` types in `tools:` frontmatter dropping all but last entry." — Changelog v2.1.147 (2026-05-21)

### What this means in practice

When the docs say "X is not supported for plugin-shipped agents," read it carefully. The scope of "for plugin-shipped agents" is **for agent frontmatter only**. The same capability is usually supported at the plugin level, where the user can audit it.

| Desired behavior | Wrong location | Right location |
|---|---|---|
| Auto-approve plugin-internal Bash | Per-agent `permissionMode: acceptEdits` | Plugin-level `PreToolUse` hook with allowlist |
| Validate file naming on Write | Per-agent `hooks: Write -> validate.py` | Plugin-level `PreToolUse` hook with `matcher: "Write\|Edit"` |
| Provide DB connectivity | Per-agent `mcpServers: ...` | Plugin-level `mcpServers` in `plugin.json` |
| Background-friendly write permission | Per-agent `permissionMode:` | `mode: "acceptEdits"` at the Task call site (parent owns) |

### The takeaway

The boundary isn't a limit on what plugins can do — it's a relocation of where things are declared. Push declarations into `plugin.json` (where users can audit them) and into call-site parameters (where parent agents explicitly grant privilege). Everything you thought was forbidden because of "not supported in plugin agents" is probably supported elsewhere.

---

## 7. State, Reference Materials, and Where They Live

The plugin cache lives at `~/.claude/plugins/cache/<id>/`. The orchestrator main thread can read files anywhere on disk. **Background subagents cannot read plugin-cache paths** — they run with restricted filesystem permissions and are limited to the project's working directory plus `additionalDirectories`. (Fabric N14.)

This means any reference file, template, risk catalog, style guide, or read-only resource that a background subagent needs must live inside the project's working directory by the time the subagent fires.

The plugin-cache convention from the plugins reference:

> "For security and verification purposes, Claude Code copies *marketplace* plugins to the user's local **plugin cache** (`~/.claude/plugins/cache`) rather than using them in-place." — https://code.claude.com/docs/en/plugins-reference

### Three patterns to make references reachable

**Pattern 1: Copy at scaffold time (best for stable, larger files).** The project initializer copies `${CLAUDE_PLUGIN_ROOT}/reference/*` into `<project>/6 - Agentic Resources/reference/` at scaffold time. All later subagents reference the project-local path. The reference content travels with the project, so it's available even after the plugin is uninstalled, and subagents can read it freely.

**Pattern 2: Embed content in subagent prompt (best for tiny content).** The orchestrator reads the reference file itself (orchestrator permissions are unrestricted), then embeds the content into the subagent's prompt string. Good when references are small (a few hundred lines) and you want a single source of truth.

**Pattern 3: Foreground subagent (escape hatch).** Foreground subagents inherit the orchestrator's permission scope and CAN read plugin-cache paths. But this defeats parallel fan-out designs and ties up the orchestrator's interactive channel. Use only when patterns 1 and 2 don't apply.

### Stage ordering matters

If the project initializer runs at Stage 7 but Stages 2–6 write into the project structure, the folder layout is created ad-hoc by `cp`/`mv`'s parent-directory auto-creation. The result is duplicate folder prefixes, conflicting numbering, and stages that write into folders other stages haven't planned for.

The rule: **the scaffolder runs before any other stage writes to the project.** Move project initialization to the earliest possible stage (right after the discovery Q&A, before any export, profile, or analysis call). Fabric originally had its scaffolder at Stage 7; the N14 fix moved it to Stage 2. (Fabric N14.)

### Validate copies with a sentinel file

`fabric-project-initializer`'s first version checked `if dir.exists()` after copying references. The copy logic had a bug that wrote two placeholder stub files; the stubs satisfied `dir.exists()`; the actual reference content was missing; the downstream risk-scan stage hard-failed.

The fix: validate the resolved directory by the presence of a known sentinel file (`m-conversion-risk-catalog.md`), not by `dir.exists()`. A half-populated or stub folder is rejected rather than silently accepted. (Fabric N16.)

### Path conventions

| Use | Where |
|---|---|
| `${CLAUDE_PLUGIN_ROOT}` | `plugin.json` mcpServers args, hook commands, agent body Bash, SKILL.md body Bash |
| `${CLAUDE_SKILL_DIR}` | Inside SKILL.md when referencing the skill's own resources (more idiomatic than `${CLAUDE_PLUGIN_ROOT}/skills/<name>/`) |
| `$HOME/.claude/skills/<name>/...` | **Never** — works only in dev when you have a standalone copy of the skill; points nowhere on install (DBT F7) |
| Hardcoded relative fallback | **Never** — env vars don't reliably propagate into subprocesses; use ancestor-walk on `Path(__file__).resolve().parents` instead (Fabric N16) |
| `${CLAUDE_PLUGIN_OPTION_<KEY>}` | Plugin script subprocess env vars (the bare name your script reads MUST be remapped from this prefixed name; see Section 4, DBT F5) |

### Open issue alert: `CLAUDE_PLUGIN_ROOT` propagation bugs

Three GitHub issues are open as of v2.1.150 documenting inconsistent `CLAUDE_PLUGIN_ROOT` behavior — Fabric N16 (script ancestor walk + sentinel file) is the defensive pattern against these:

- Issue #27145: "`CLAUDE_PLUGIN_ROOT` environment variable not set for SessionStart hooks." — https://github.com/anthropics/claude-code/issues/27145
- Issue #36585: "`CLAUDE_PLUGIN_ROOT` not passed to UserPromptSubmit hooks in plugin system." — https://github.com/anthropics/claude-code/issues/36585
- Issue #38699: *"For plugins installed from local marketplaces, `CLAUDE_PLUGIN_ROOT` is set to different values depending on context — hooks receive the source directory while the agent's shell environment receives the cache/install directory."* — https://github.com/anthropics/claude-code/issues/38699

---

## 8. Orchestrator Topology

Claude Code enforces a one-level subagent hierarchy:

> "Subagents cannot spawn other subagents, so `Agent(agent_type)` has no effect in subagent definitions."

This single rule determines almost everything about how orchestrators must be designed and invoked.

### Orchestrators only work as main-thread agents

The orchestrator must be launched directly:

```bash
claude --agent my-plugin:orchestrator:orchestrator "Build a pipeline"
```

If the orchestrator is invoked any other way — via auto-delegation from a regular Claude session, via `/agents` picker from inside an existing session, via `Task()` from another agent — it becomes a subagent. Its `Agent(...)` tool is then inert. Its `Task()` calls silently no-op. The pipeline stalls with no error. (DBT F4.)

v2.1.116 confirms the main-thread-only constraint for agent-declared hooks:

> "Agent `hooks:` fire as main-thread" — Changelog v2.1.116

v2.1.143 fixed a related discovery bug:

> "Fixed `--agent` not finding plugin-contributed agents." — Changelog v2.1.143

### Slash commands cannot host orchestrators

A slash command runs inside an existing Claude session. If the slash command spawns the orchestrator via `Task(...)`, the orchestrator is a subagent (see above) and cannot delegate. **Never make an orchestrator a slash command.** (Fabric N11.)

Slash commands CAN still trigger leaf agents that don't fan out — a `/run-tests` slash command that invokes a single test-runner specialist is fine. Just not orchestrators.

### Invocation guidance must be unmissable

Both plugins' READMEs document the canonical `claude --agent <plugin>:<subdir>:<name>` launch prominently. Burying it in the agent body or in a "see also" link is a guaranteed support burden. Print the exact command in the marketplace listing, in the post-install message, in the project initializer output — wherever the user might land.

### Trust-but-verify after every fan-out

Background subagent envelopes are not reliable evidence of success. The Fabric plugin caught a class of bugs where 4 bronze builders reported `status: 'success'` with valid `notebook_path` values — and then no `.ipynb` files existed on disk. The worktree-remove hook deleted them milliseconds after the envelope was returned. (Fabric N8.)

The fix: after every fan-out, the orchestrator `ls`s the claimed output paths and confirms the files actually exist on disk. Missing file → halt and surface as a deviation. This belongs in every orchestrator's stage logic after every background fan-out, not just the cases that have failed historically.

This pattern aligns with Anthropic's explicit framing of the agent loop:

> "gather context → take action → verify work → repeat" — Building agents with the Claude Agent SDK (2025-09-29) — https://claude.com/blog/building-agents-with-the-claude-agent-sdk

### Worktree isolation: only when safe

Worktree isolation is safe only when (a) the agent writes files that are already git-tracked AND (b) the `WorktreeRemove` hook commits and merges back to the main worktree before `git worktree remove --force`. The default dbt-plugin hooks do NOT merge back — they assume the agent committed the work itself, which is true for dbt models-as-source-controlled-text but false for any agent writing new artifacts (notebooks, build outputs, generated docs).

If the agent creates new files, either (a) add a merge-back step to `WorktreeRemove` before `git worktree remove`, or (b) drop worktree isolation entirely and use unique per-agent filenames (`nb_{layer}_{query}.ipynb`) to avoid collisions. Fabric chose option (b) after N8 surfaced. (Fabric N8.)

---

## 9. User Interaction Budget

A plugin's user-interaction budget should be 2–3 touch points across the entire pipeline. Each touch point costs user attention, breaks parallelism, and creates a place the pipeline can pause indefinitely waiting for input. Minimize them.

### The standard shape

| # | Purpose | Tool |
|---|---|---|
| 1 | Discovery Q&A (gather requirements / config) | `AskUserQuestion` in orchestrator main thread |
| 2 | Plan / refactor decisions (after analysis, before build) | `AskUserQuestion` in orchestrator main thread |
| 3 | Approval gate (review plan, choose proceed/revise/abort) | `AskUserQuestion` in orchestrator main thread |

The DBT plugin runs at 2 touch points (discovery + approval). The Fabric plugin runs at 3 (config + refactor decisions + approval) because Fabric needs runtime workspace IDs that aren't available at install time.

### Skip `AskUserQuestion` outside these gates

If a specialist subagent wants to ask the user something, restructure the pipeline. `AskUserQuestion` doesn't work in subagents (Section 4, Fabric N15). Use the three-step parent-owned pattern:

1. **Subagent in analyze mode** reads the inputs, emits a JSON envelope (`refactor-questions.json`) of questions and their options.
2. **Parent (orchestrator)** reads the envelope, calls `AskUserQuestion` with the questions array, writes the user's answers to a second envelope (`refactor-answers.json`).
3. **Subagent in write mode** reads the answers envelope and produces the side-effecting outputs deterministically.

Both subagent spawns run in background; neither needs `AskUserQuestion`. The parent owns interactive Q&A, the subagents are purely deterministic specialists. (Fabric N15.)

v2.1.147 fixed a related main-thread suppression bug:

> "Fixed auto mode suppressing `AskUserQuestion` when user or skill explicitly relies on it." — Changelog v2.1.147

### Plan-mode approval: use `AskUserQuestion` instead of `EnterPlanMode`

`EnterPlanMode` must be explicitly granted in the agent's `tools:` frontmatter. If it's missing, calls silently no-op — there's no tool-not-available error visible to the agent, just a quiet skip past the approval gate.

Use `AskUserQuestion(Approve/Revise/Abort)` instead. It's already in most agents' default toolset, it doesn't depend on a tool that might not be available, and it gives you explicit branching (`Revise` can loop back to a previous stage; `Abort` can write a reason file and exit cleanly). (Fabric N10.)

Community evidence of plan-mode leakage in subagents (Issue #43777):

> "When plan mode is active (via Shift+Tab), using the Agent tool spawns subagents that are not subject to the plan mode constraint. These subagents can freely make file edits and run non-readonly tools, effectively bypassing plan mode entirely." — https://github.com/anthropics/claude-code/issues/43777

### Dynamic question selection

Don't ask hardcoded "always ask 5 questions" sets when most users need a subset. Read the upstream analysis first and ask only questions that apply:

- Excel question only if Excel sources were detected
- Combine Files question only if helper queries were found
- AzureStorage question only if blob URIs were detected

Cap at 4 questions per `AskUserQuestion` call. Users with simple inputs get the minimum (1–2 questions). Users with complex inputs get all relevant decisions. (Fabric N4.)

---

## 10. Testing Strategy (Fresh Install Is the Only Truth)

Every fix in this document was applied because something that worked in dev mode silently broke on a fresh install. The reverse also holds: many "this can't possibly work" claims based on GitHub issues turned out to be wrong when tested on a fresh install. The fresh install is the source of truth.

### What "fresh install" means

A clean machine that has never seen your repo, OR a wipe of `~/.claude/plugins/cache/` followed by `/plugin marketplace add ...` and `/plugin install ...`. Anything less and you're still partially testing dev artifacts.

Dev shortcuts hide:

- The 3-part namespace (you have bare-name agents in `~/.claude/agents/`)
- Stripped frontmatter fields (your standalone `.md` files still respect them)
- Plugin-cache vs project-local paths (your `$HOME/.claude/skills/` copies still exist)
- Background subagent permission gaps (you can answer the prompts interactively)
- `userConfig` prefix env vars (you have the bare names exported)
- The slash-command-makes-orchestrator-a-subagent bug (you launch from a fresh shell)

### Verification checklist

Before declaring a release "done," run all of these on a fresh install:

- `/agents` picker shows the registered agent names. Check that subdirectory agents show the 3-part form.
- `/skills` picker shows 2-part skill names marked "locked by plugin."
- `/mcp` lists the bundled MCP servers and they appear connected.
- `/plugin` shows the plugin enabled with the expected version.
- End-to-end run with `--sample --dry-run` produces the expected output files in the expected paths. Bundle sample inputs in `examples/` so this is possible without user data.
- Each stage's output files actually exist on disk after the orchestrator claims that stage succeeded.
- The `userConfig` prompt fired during install. If it didn't, document the manual `settings.json` fallback in the README.

### Don't trust GitHub issues

The DBT plugin team spent time designing around the impossibility of plugin-to-plugin subagent delegation — an Anthropic feature request was closed "Not Planned" on 2026-02-27, multiple open issues described the same architectural wall, no public plugin appeared to do it. Then a fresh-install test showed it working on the first try. (DBT F6.)

The issue in question (#19276):

> "The `Task()` tool currently only recognizes 3 hardcoded built-in subagents... Any attempt to use a custom agent fails: `Agent type 'custom-agent' not found. Available agents: Bash, general-purpose, plan, explore, ...`" — https://github.com/anthropics/claude-code/issues/19276

Yet v2.1.143 explicitly "Fixed `--agent` not finding plugin-contributed agents" and v2.1.140 "Improved Agent tool `subagent_type` matching (case/separator insensitive)" — proving the path is alive and being maintained.

Lessons: docs are incomplete. Issue threads go stale. "Not Planned" sometimes becomes "quietly works." The only reliable signal is your actual plugin running on a fresh install today. That said: features built on top of "not officially supported" behavior can regress in any future release. Document a fallback architecture before you ship, and run a smoke test in CI on every release.

### Anthropic's own postmortem confirms even rigorous evals miss things

> "The changes it introduced made it past multiple human and automated code reviews, as well as unit tests, end-to-end tests, automated verification, and dogfooding." — April 23 postmortem (2026-04-23) — https://www.anthropic.com/engineering/april-23-postmortem

If Anthropic's internal pipeline can ship a default-reasoning-effort regression past every gate, your plugin definitely needs a fresh-install smoke test that exercises the end-to-end happy path.

---

## 11. Observability & Failure Surfacing

Background subagents fail silently by default. The envelope says `status: 'success'`. Nothing actually happened on disk. The orchestrator advances. The user finds out at the end of a 30-stage pipeline that Stage 2 never produced anything.

Five mitigations, applied in layers:

### Mitigation 1: Trust-but-verify after every fan-out

After every Task fan-out, the orchestrator `ls`s the claimed output paths and confirms files exist on disk. Missing file → halt with a deviation report. This catches:

- Background subagents whose Bash was silently blocked by the permission layer
- Worktree-remove hooks deleting output files before the orchestrator sees them
- File writes that were intercepted by a validate-structure hook with `decision: block`
- Subagents that "succeeded" but pointed at a different path than expected

This pattern belongs in every orchestrator's stage logic after every background fan-out. (Fabric N8.)

Best Practices doc reinforces it:

> "Give Claude a way to verify its work. Include tests, screenshots, or expected outputs so Claude can check itself. This is the single highest-leverage thing you can do." — https://code.claude.com/docs/en/best-practices

### Mitigation 2: Structured envelope contract

Every subagent returns a JSON envelope with a strict schema:

```json
{
  "status": "success" | "error",
  "deviation": "<reason>" | null,
  "paths": ["<path1>", "<path2>", ...],
  "counts": {"queries_converted": 7, "high_risk": 2, "..."},
  "notes": "..."
}
```

The orchestrator validates the envelope shape and the path existence before advancing. Subagents that fail to return a well-formed envelope are treated as failures, not successes. (Fabric inherits across all stages.)

### Mitigation 3: `statusMessage` on every hook

Hooks declared in `plugin.json` should include `statusMessage` so the spinner tells the user what's running:

```json
{
  "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/approve-plugin-bash.py",
  "statusMessage": "Checking plugin Bash allowlist..."
}
```

Without the message, the user sees a spinner with no context and assumes the system is stuck. With it, they see "Checking plugin Bash allowlist..." and know a hook is doing real work.

### Mitigation 4: SessionStart config-check hook

A SessionStart hook that detects missing `userConfig` values, missing prerequisites (`fab`, `az`, ODBC driver), or stale auth tokens and surfaces a remediation message at session open. Both plugins ship variants of this; Fabric's `hooks/session-start-config-check.py` is the canonical example.

This is the mitigation for the DBT plugin's "userConfig prompt didn't fire on install" problem (DBT F5 Problem B) — even if the install-time prompt never shows, the SessionStart hook catches the missing config the first time the user opens a session. Issue #39455 confirms the install-prompt skip is a known unresolved bug:

> Issue #39455: "Plugin userConfig values not prompted on enable." — https://github.com/anthropics/claude-code/issues/39455

### Mitigation 5: Write a `plugin_learnings.md` as you build

Capture every silent failure with its root cause, fix, and reusable rule. The doc becomes:

- A regression checklist for future releases
- Source material for documentation, talks, and onboarding
- A search-grep target when a new contributor hits a confusing symptom
- A signal of what kinds of bugs the plugin has historically been vulnerable to

Both shipped plugins have such docs (`_Documentation/plugin_learnings.md`) and they were the primary source for this best-practices file.

### Bonus: v2.1.149 `/usage` per-category breakdown

> "`/usage` now shows per-category breakdown (skills, subagents, plugins, per-MCP-server cost)." — Changelog v2.1.149 (2026-05-23)

Use this to right-size your plugin's context budget. If a single skill or MCP server in your plugin dominates `/usage`, that's a refactor signal.

---

## 12. Release Engineering for Plugins

Plugins are shipped artifacts, not sandboxes. Users can't patch a plugin in place — the cached files are marked "locked by plugin," edits are overwritten on update, and the user's only escape hatches are (a) uninstall, (b) fork, or (c) wait for an update. (DBT F8.)

This raises the bar for release quality.

### Pre-shipment audit

Both plugins ship a `tests/preshipment_audit.py` script that runs a battery of gates before a release tag.

> **Canonical gate catalog:** See `how-to-create-plugins.md` § 14 ("Pre-Shipment Audit") for the full 7-gate table with exact check definitions and sample output. The Fabric plugin runs all 7; the DBT plugin runs 6 (omitting `notebook_extension`). Every new finding adds a gate.

### Semver in `plugin.json`

Bump `version` on every release. The Fabric plugin runs `0.5.0` after 17 numbered findings; the DBT plugin runs `1.0.0` after 10 findings. Semver doesn't need to be aspirational — minor bumps for new findings/refactors are fine, major bumps for breaking the orchestrator's public command shape.

v2.1.119 also documented that plugin auto-update respects semver:

> "Plugin auto-update respects constraints" (semver). — Changelog v2.1.119

### Marketplace update behavior

For most plugin changes (agents, skills, hook scripts, reference materials, manifests), `/reload-plugins` propagates the update immediately. For MCP server changes (new server, changed args, changed env), a full Claude Code restart is needed because MCP servers are started once per session.

Document this in the changelog: "MCP server changed — restart Claude Code after updating" vs "Agents updated — `/reload-plugins` is sufficient."

### Plugin dependency declarations (v2.1.143)

> "Added plugin dependency enforcement: `claude plugin disable` refuses when another plugin depends on target; `claude plugin enable` force-enables transitive dependencies." — Changelog v2.1.143

If you publish multiple plugins with shared infrastructure (e.g. a shared `sql-connection` skill), declare dependencies explicitly so users don't accidentally break a working stack by disabling a dependency.

### Backlog and issues per the user's global convention

Maintain `_Plan/Backlog.md` (forward-looking planned work) and `_Plan/Issues.md` (known bugs, empirical verification needs, architectural risks) at the plugin repo level. Both plugins follow this convention. Every silent failure goes in `Issues.md` before (or in the same commit as) the fix. Undocumented fixes are invisible to future contributors.

---

## 13. Top 16 Anti-Patterns to Avoid

| # | Anti-Pattern | Why It Breaks | Fix | Source |
|---|---|---|---|---|
| 1 | Bare-name agent references (`subagent_type: "business-analyst"`) | Namespace lookup resolves to nothing on install | Use the 3-part form `<plugin>:<subdir>:<name>` | DBT F1 |
| 2 | `permissionMode: acceptEdits` in plugin agent frontmatter | Silently stripped at install time | Set `mode: "acceptEdits"` at the Task call site | DBT F2 |
| 3 | `hooks:` / `mcpServers:` in plugin agent frontmatter | Silently stripped at install time | Push declarations to `plugin.json` | DBT F2 |
| 4 | Background subagent without `mode: "acceptEdits"` | Stalls on first `Write` — no interactive channel for permission prompt | Always set `mode` at the call site for `run_in_background: true` | DBT F3 |
| 5 | Orchestrator launched as a slash command | Becomes a subagent, can't delegate, pipeline stalls | Launch with `claude --agent <plugin>:<subdir>:<name>` | Fabric N11 |
| 6 | `AskUserQuestion` in a subagent | Not supported regardless of fg/bg — falls back to defaults | Three-step parent-owned pattern (subagent emits Q envelope, parent asks, subagent consumes A envelope) | Fabric N15 |
| 7 | `EnterPlanMode` without it in `tools:` frontmatter | Silent no-op — no error visible | Use `AskUserQuestion(Approve/Revise/Abort)` instead | Fabric N10 |
| 8 | `$HOME/.claude/skills/<name>/scripts/...` in agent or SKILL body | Points nowhere on install — Python exits with file-not-found | Use `${CLAUDE_PLUGIN_ROOT}/skills/<name>/scripts/...` | DBT F7 |
| 9 | Backslashes inside script paths (`\scripts\foo.py`) | Git Bash interprets `\s`, `\f` as escape sequences | Forward slashes always, even on Windows | DBT F7 |
| 10 | Skill scripts reading bare env var names (`SQL_SERVER`, `FABRIC_TENANT_ID`) | `userConfig` values arrive as `CLAUDE_PLUGIN_OPTION_<KEY>` | Add `_load_plugin_userconfig_env()` helper before argparse | DBT F5 |
| 11 | Assuming the `userConfig` install-time prompt is reliable | Sometimes silently doesn't fire on install/update | SessionStart hook as fallback; README documents manual `settings.json` fallback | DBT F5 Problem B |
| 12 | Worktree hooks on agents writing untracked files | `git worktree remove --force` deletes the files before merge | Drop worktree isolation, use unique per-agent filenames OR add merge-back to `WorktreeRemove` | Fabric N8 |
| 13 | Background subagent reading plugin-cache files | Permission layer blocks the read | Copy refs to project at scaffold OR embed inline in prompt | Fabric N14 |
| 14 | Compound Bash commands (`&&`, `\|\|`, `;`, `\|`, subshells, `$(...)`) | Permission layer fall-through; auto-approval lost; stalls in background | Atomic commands only; sequential logic in LLM text | DBT F9 Round 2 |
| 15 | `.py` notebook output for Fabric | Fabric treats `.py` as a single mega-cell, breaks deployment | `.ipynb` Jupyter JSON; runtime hook blocks `.py` writes; audit gate prevents drift | Fabric N1, N17 |
| 16 | PowerShell scripts without UTF-8 BOM | PS 5.1 falls back to Windows-1252; em-dashes/smart quotes break string parsing | Write generated `.ps1` as `utf-8-sig`; audit static scripts for non-ASCII; prefer pure 7-bit ASCII | Fabric N13 |

### Honorable mentions (worth knowing about but already covered above)

- Inventing CLI flags that don't exist on your own scripts (DBT F9 Round 2.5)
- Trusting agent envelopes without verifying side effects (Fabric N8)
- Treating "Not Planned" as "Not Possible" without testing on a fresh install (DBT F6)
- Scaffolding the project structure after stages that write into it (Fabric N14)
- Validating reference copies with `dir.exists()` instead of a sentinel file (Fabric N16)

---

## 14. Themes

The findings cluster around seven high-level themes worth internalizing as design principles. Each one applies beyond Claude Code plugins specifically — they generalize to any system with permission boundaries, audit surfaces, and delegated execution.

### 1. Agentic pipelines need explicit permission contracts

Foreground and background subagents have fundamentally different permission semantics. The permission mode names (`acceptEdits`, `bypassPermissions`) are marketing terms, not specifications. The actual behavior is in the docs, but you have to read the docs, not the names. Plugins strip the privilege fields you'd naturally use to paper over the difference, and the strip is silent. The correct architecture is to declare privilege at the manifest level (auditable) and pass it explicitly at the spawn site (parent-owned), not via runtime metadata.

### 2. Namespacing is a silent correctness issue

The registered name of a plugin component depends on its directory depth and the docs example only covers the flat case. Subdirectory agents get an extra namespace segment. Skills are always flat. The agent `skills:` field uses a different format than `subagent_type:`. None of this is fully documented; all of it fails silently when wrong. The only fix is to verify in the `/agents` and `/skills` pickers on a fresh install and ship an audit gate that grep-validates every cross-reference.

### 3. Testing on a fresh install is non-optional

Dev mode hides the bugs that matter most. GitHub issues go stale. "Not Planned" feature requests sometimes quietly become "works today." The behavior of your actual plugin running on a fresh install today is the only reliable signal — and you have to test it on a clean machine, not on the dev machine where the standalone copies still exist.

### 4. Plugin frontmatter is a security boundary

Claude Code's strip-elevated-fields-from-agents policy isn't a bug — it's the right default. Privileged declarations belong in the audited manifest, not in spawn-time metadata. Understanding where the boundary is lets you build capabilities that initially look forbidden (PreToolUse hooks at plugin level, MCP servers, lifecycle event handlers) by relocating declarations to the right surface.

### 5. "Not Planned" doesn't always mean unavailable

Some features work today despite being explicitly rejected in the issue tracker. That's a gift — but it's also a stability risk. If your plugin depends on a feature Anthropic has declined to support, you need a documented fallback architecture and a smoke test in CI that will catch regression. Don't assume the platform won't move out from under you.

### 6. Match the tool's grain

Claude Code's permission layer, hook system, and `acceptEdits` mode are all designed around the assumption that each Bash tool call is a single atomic operation. The moment you introduce compound expressions, every layer has to do extra work to figure out what you meant — and at least one layer eventually gets it slightly wrong. The same principle generalizes beyond Claude Code: any time you're integrating with a permission system, an audit log, a rule engine, or a task queue, single-operation atoms are the grain the system is designed around. Compound operations require the system to parse your intent, and parsing intent is where bugs live.

### 7. Belt and suspenders

The atomic-Bash rule lives in the orchestrator body, every specialist body, the project CLAUDE.md template, the global `~/.claude/CLAUDE.md`, the plugin's own learnings doc, and the pre-shipment audit. The `.ipynb` rule lives in the agent body, the validate-structure hook, the pre-shipment audit, and the orchestrator's stage prompts. Each layer catches a different drift mode. Each audience reads a different file. Skip any one and that audience eventually goes off-script. Duplication across audience-specific surfaces is cheap; missing the rule in any one of them is expensive.

---

## 15. Source Validation: Official Corroborations & Open Questions

The findings throughout this doc came from empirical work on two shipped plugins. Cross-checking against the official Claude Code changelog, the plugins reference, the Agent SDK announcement, and triangulated community-issue reporting validates most of them — but a few are still undocumented in any official release note. Use this table to know which rules in this doc rest on Anthropic-published behavior vs. which rest on our own observations.

Every quote below is verbatim from primary sources, with date and URL inline.

| Finding (our shorthand) | Official Status | Verbatim Source |
|---|---|---|
| F1 — 3-part agent namespace (`plugin:subdir:name`) | **No public release note found.** Our observation, verified on fresh install. | Plugins reference (https://code.claude.com/docs/en/plugins-reference) uses `name@marketplace` (2-part) for `${CLAUDE_PLUGIN_DATA}` IDs: *"The `${CLAUDE_PLUGIN_DATA}` directory resolves to `~/.claude/plugins/data/{id}/`, where `{id}` is the plugin identifier with characters outside `a-z, A-Z, 0-9, _, -` replaced by `-`. For a plugin installed as `formatter@my-marketplace`, the directory is `~/.claude/plugins/data/formatter-my-marketplace/`."* The 3-part form for subdirectory-based agents is consistent with v2.1.145's `/plugin` Discover panes listing "commands, agents, skills, hooks, and MCP/LSP servers" per plugin, but the exact format was never announced. |
| F2 — frontmatter stripping for `hooks`/`mcpServers`/`permissionMode` | **Corroborated.** | Plugins reference: *"For security reasons, `hooks`, `mcpServers`, and `permissionMode` are not supported for plugin-shipped agents."* — https://code.claude.com/docs/en/plugins-reference. Plugin-shipped agents support frontmatter fields *"`name`, `description`, `model`, `effort`, `maxTurns`, `tools`, `disallowedTools`, `skills`, `memory`, `background`, and `isolation`. The only valid `isolation` value is `'worktree'`."* (2026-05-21 — v2.1.147 Changelog): *"Fixed plugin agents declaring multiple `Agent(...)` types in `tools:` frontmatter dropping all but last entry."* — confirms a related silent-drop bug, finally fixed. Source: https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md |
| F3 — background subagents cannot answer permission prompts | **Corroborated by behavior, not by a release note.** | (2026-05-15 — v2.1.143 Changelog): *"Added `worktree.bgIsolation: 'none'` setting for background sessions to edit working copy directly without `EnterWorktree`."* — Anthropic acknowledged the worktree model needed an escape hatch for background sessions. |
| F4 — orchestrators only delegate from main thread | **Corroborated.** | (2026-Earlier — v2.1.116 Changelog): *"Agent `hooks:` fire as main-thread"* — agent-declared hooks fire only in the agent's main thread, not in spawned children. Same shape applies to delegation. (2026-05-15 — v2.1.143): *"Fixed `--agent` not finding plugin-contributed agents."* |
| F5 — `CLAUDE_PLUGIN_OPTION_<KEY>` env-var naming | **Corroborated.** | Plugins reference: *"The `userConfig` field declares values that Claude Code prompts the user for when the plugin is enabled. Use this instead of requiring users to hand-edit `settings.json`."* And: *"Each value is available for substitution as `${user_config.KEY}` in MCP and LSP server configs, hook commands, and monitor commands. Non-sensitive values can also be substituted in skill and agent content. All values are exported to plugin subprocesses as `CLAUDE_PLUGIN_OPTION_<KEY>` environment variables. Sensitive values are stored in the system keychain."* — https://code.claude.com/docs/en/plugins-reference. (2026-Earlier — v2.1.119): *"Fixed plugin user_config blanks."* |
| F5 Problem B — userConfig prompt may not fire on install | **Corroborated as an open issue.** | GitHub Issue #39455: "Plugin userConfig values not prompted on enable." — https://github.com/anthropics/claude-code/issues/39455. Confirms the silent-skip we observed. |
| F6 — "Not Planned" feature delegation actually works | **Corroborated.** | Issue #19276 ("Custom Subagent Support in Task Tool") — closed as "Not Planned" with labels `area:core`, `area:tools`, `enhancement`, `stale`. Quote from issue body: *"The `Task()` tool currently only recognizes 3 hardcoded built-in subagents... Any attempt to use a custom agent fails: `Agent type 'custom-agent' not found. Available agents: Bash, general-purpose, plan, explore, ...`"* — https://github.com/anthropics/claude-code/issues/19276. Despite this, (2026-05-15 — v2.1.143): *"Fixed `--agent` not finding plugin-contributed agents"* and (v2.1.140): *"Improved Agent tool `subagent_type` matching (case/separator insensitive)"* prove the discovery path is alive and being maintained. The stability risk this implies remains real. |
| F7 — `${CLAUDE_PLUGIN_ROOT}` for plugin-internal paths | **Corroborated, with known bugs.** | Plugins reference documents `${CLAUDE_PLUGIN_ROOT}` substitution. Three open issues flag inconsistent propagation: Issue #27145 (https://github.com/anthropics/claude-code/issues/27145): *"`CLAUDE_PLUGIN_ROOT` environment variable not set for SessionStart hooks."* Issue #36585 (https://github.com/anthropics/claude-code/issues/36585): *"`CLAUDE_PLUGIN_ROOT` not passed to UserPromptSubmit hooks in plugin system."* Issue #38699 (https://github.com/anthropics/claude-code/issues/38699): *"For plugins installed from local marketplaces, `CLAUDE_PLUGIN_ROOT` is set to different values depending on context — hooks receive the source directory while the agent's shell environment receives the cache/install directory."* — exactly the kind of bug N16 (script ancestor walk + sentinel file) defends against. |
| F8 — plugin skills 2-part namespace, "locked by plugin" | **Corroborated.** | Plugins reference: *"Plugin skills use a `plugin-name:skill-name` namespace, so they cannot conflict with other levels."* — https://code.claude.com/docs/en/plugins-reference. Marketplace cache policy: *"For security and verification purposes, Claude Code copies marketplace plugins to the user's local plugin cache (`~/.claude/plugins/cache`) rather than using them in-place."* (2026-05-14 — v2.1.142): *"Plugins with root-level `SKILL.md` and no `skills/` subdirectory now surfaced as skill"* — flat-layout supported. |
| F9 — PreToolUse Bash allowlist hook unblocks background subagents | **Corroborated by docs.** | Hooks reference: *"PreToolUse hooks run before the permission prompt. The hook output can deny the tool call, force a prompt, or skip the prompt to let the call proceed."* — https://code.claude.com/docs/en/hooks. The narrow allowlist pattern is our adaptation; the underlying mechanism is documented. |
| F10 — dev vs installed behavior diverges | **Implicitly corroborated.** | Every other entry in this table is an example of the dev/install divergence. The plugin reference's "For security reasons..." language plus the SDK-mode quirks (v2.1.118 "Fixed SDK plugin reload serial," v2.1.141 "Fixed early OTel spans dropped in SDK mode," v2.1.147 "Fixed uncaught exception at streaming session end via Agent SDK") all support the meta-claim that dev shortcuts hide install-time bugs. |
| N1 — Fabric `.ipynb`, not `.py` | Anthropic plugin spec is silent on this — it's a Fabric-API correctness rule, not a Claude Code rule. | Our observation. Belt-and-suspenders enforcement (agent body + hook + audit) is the right shape because the platform won't catch it for you. |
| N8 — worktree hooks destroy newly-written files | **Corroborated.** | (2026-05-15 — v2.1.143 Changelog): *"Added `worktree.bgIsolation: 'none'` setting for background sessions to edit working copy directly without `EnterWorktree`."* Anthropic acknowledged the worktree model needed an escape hatch. |
| N9 + N10 — foreground subagent and `EnterPlanMode` rules | **Partially corroborated.** | Community-confirmed in Issues #24072, #43777, #34592: *"The subagent tool filter strips `AskUserQuestion` for in-process teammates and also includes `TaskOutput`, `ExitPlanMode`, `EnterPlanMode`, `Task`, and `TaskStop` in the exclusion set."* — https://github.com/anthropics/claude-code/issues/34592. Plan-mode subagent leakage in Issue #43777: *"When plan mode is active (via Shift+Tab), using the Agent tool spawns subagents that are not subject to the plan mode constraint. These subagents can freely make file edits and run non-readonly tools, effectively bypassing plan mode entirely."* — https://github.com/anthropics/claude-code/issues/43777. (2026-05-21 — v2.1.147 Changelog): *"Fixed auto mode suppressing `AskUserQuestion` when user or skill explicitly relies on it"* — fix was for *auto mode in the main thread*, not for subagents. The subagent restriction remains undocumented as intentional. |
| N11 — slash commands can't host orchestrators | **No public release note found.** | Our observation. Consistent with the subagent-hierarchy rule (a subagent can't spawn subagents), but the slash-command path specifically has never been documented. |
| N12 — corporate TLS interception breaks Python-based CLIs | **Not Claude-Code-specific.** | Industry-wide. Worth surfacing in plugin READMEs because every plugin that wraps `az`, `fab`, `ms-fabric-cli`, or anything using `requests` will hit it on managed-AV machines. |
| N13 — PowerShell UTF-8 BOM requirement | **Not Claude-Code-specific.** | PS 5.1 platform behavior. Same enforcement bar as N12 — bake it into the plugin's Python-writing-PowerShell layer. |
| N14 — background subagents cannot read plugin-cache paths | **Corroborated by behavior, not by a release note.** | Background subagents run with restricted filesystem permissions in worktree-isolated contexts. v2.1.143's bgIsolation escape hatch implies Anthropic acknowledges this is constraining; the explicit "subagents can't read plugin cache" rule is our finding. |
| N15 — `AskUserQuestion` not in any subagent | **Corroborated.** | Issue #34592: *"`AskUserQuestion` is completely absent from sub-agent environments — not in direct tools, deferred tools, or `ToolSearch` results."* — https://github.com/anthropics/claude-code/issues/34592. Issue #18721 specifically asks for a "missing warning for AskUserQuestion subagent limitation" — https://github.com/anthropics/claude-code/issues/18721. Anthropic's Agent SDK user-input docs explicitly state: *"Subagents: `AskUserQuestion` is not currently available in subagents spawned via the Agent tool."* |
| N16 — resolve bundled resources via ancestor walk + sentinel | **No release note.** | Defensive coding pattern we developed to handle the inconsistent-env-var-propagation bug surfaced by Issues #27145, #36585, #38699 (cited under F7). Will become unnecessary if Anthropic ever fully fixes `CLAUDE_PLUGIN_ROOT` propagation; until then, ship the helper. |
| N17 — builder pollution (`.py` next to `.ipynb`, JSON envelopes on disk, etc.) | **No release note.** | Plugin-prompt-discipline issue, not a platform bug. Each builder needs explicit "don't pollute" instructions plus a pre-shipment audit gate. |

### What the changelog adds beyond our findings

Features and fixes worth designing around that didn't appear in our two plugins, with the verbatim release-note quotes:

- **v2.1.149 (2026-05-23) — `/usage` per-category breakdown.** *"`/usage` now shows per-category breakdown (skills, subagents, plugins, per-MCP-server cost)."* Token cost now shows separately for skills, subagents, plugins, per-MCP-server. Use this to right-size plugin context budget.
- **v2.1.145 (2026-05-19) — `/plugin` Discover preview.** *"`/plugin` Discover/Browse screens now show plugin's commands, agents, skills, hooks, and MCP/LSP servers before installation."* Treat your plugin's component count, names, and descriptions as a marketing surface.
- **v2.1.143 (2026-05-15) — plugin dependency enforcement.** *"Added plugin dependency enforcement: `claude plugin disable` refuses when another plugin depends on target; `claude plugin enable` force-enables transitive dependencies."* If you publish multiple plugins with shared infrastructure (e.g. a shared `sql-connection` skill), declare dependencies explicitly.
- **v2.1.143 (2026-05-15) — projected context cost in browse pane.** *"Added projected context cost to `/plugin` marketplace browse pane."* A tight plugin context budget is a discoverability advantage.
- **v2.1.142 (2026-05-14) — root-level `SKILL.md` plugins.** *"Plugins with root-level `SKILL.md` and no `skills/` subdirectory now surfaced as skill."* Flat single-skill plugins now possible. Lower barrier for tiny plugins; consider when an existing skill could reach more users as its own plugin.
- **v2.1.141 (2026-05-13) — `CLAUDE_CODE_PLUGIN_PREFER_HTTPS`.** *"Added `CLAUDE_CODE_PLUGIN_PREFER_HTTPS` to clone GitHub plugin sources over HTTPS"* (vs default SSH). Managed environments without SSH access can still clone plugins. Document this in your README for corporate users.
- **v2.1.139 — `claude plugin details <name>`.** *"Added `claude plugin details <name>`."* CLI-level introspection. Plugin authors should run this against their own releases to spot regressions.
- **v2.1.139 — `init.plugin_errors` in headless mode.** *"Headless `init.plugin_errors` includes load failures."* Critical for SDK / `-p` mode deployments.
- **v2.1.133 — subagent skill discovery fix.** *"Fixed subagents not discovering skills"* — explicit fix; subagents previously failed to inherit skill context.
- **v2.1.121 — `claude plugin prune` + MCP `alwaysLoad`.** *"Added `claude plugin prune` command."* And: *"Added `alwaysLoad` option for MCP servers"* (plugin-shipped or otherwise).
- **v2.1.129 — `--plugin-url` and experimental gating.** *"Added `--plugin-url <url>` for fetching plugin archives."* And: *"Plugin manifests: `themes`/`monitors` under `experimental`"* — explicit experimental gating.
- **v2.1.119 — security and reload fixes.** *"Plugin auto-update respects constraints"* (semver). *"Security: `blockedMarketplaces` enforces patterns."* *"Fixed plugin MCP Windows spawn."* *"Fixed pre-compaction skill re-execution."* *"Fixed `/reload-plugins` disabled error."*
- **v2.1.117 — dependency chasing.** *"`plugin install` installs missing dependencies"* — first version to chase a full dependency graph.

### What Anthropic blog posts add beyond the changelog

Verbatim quotes that reinforce architectural principles in this doc. (Full quote library in Section 16.)

- **Building Effective Agents (2024-12-19):** *"The most successful implementations weren't using complex frameworks or specialized libraries. Instead, they were building with simple, composable patterns."* — https://www.anthropic.com/engineering/building-effective-agents. Maps to the orchestrator-plus-specialists pattern (Section 2).
- **Building agents with the Claude Agent SDK (2025-09-29):** *"gather context → take action → verify work → repeat."* — https://claude.com/blog/building-agents-with-the-claude-agent-sdk. Directly maps to trust-but-verify after every fan-out (Section 11).
- **Equipping agents for the real world with Agent Skills (2025-10-16):** *"Progressive disclosure is the core design principle that makes Agent Skills flexible and scalable."* — https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills. Validates the lazy-loading skill pattern.
- **How we built our multi-agent research system (2025-06-13):** *"Multi-agent systems work mainly because they help spend enough tokens to solve the problem."* — https://www.anthropic.com/engineering/multi-agent-research-system. Informs user-interaction-budget design (Section 9): if you're spending the tokens, make the deliverable demonstrably bigger.
- **Best practices for Claude Code (continuously updated):** *"Hooks run scripts automatically at specific points in Claude's workflow. Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens."* — https://code.claude.com/docs/en/best-practices. Plus the relevant guidance for plugin authors: *"Plugins bundle skills, hooks, subagents, and MCP servers into a single installable unit from the community and Anthropic."*

### Community video coverage of our findings

The expert-videos research shows that 9 of our 10 DBT findings have parallel community coverage (validating they're not idiosyncratic), but 15 of 17 Fabric findings are not covered by any major YouTube creator we surveyed — the migration-orchestrator workflow is largely original territory. The single biggest community blind spot: **the atomic-Bash rule (F9 Round 2)** is not discussed by any creator we found. This makes it the strongest "new" contribution if the user gives a conference talk. Full video index in Section 17.

### Open questions still unanswered by public sources

1. **Exact 3-part namespace format.** No public release note announces the `plugin:subdir:name` form; our observation only.
2. **History of when `CLAUDE_PLUGIN_ROOT` was first exposed.** Appears as a fact in the plugins reference with no launch bullet. Best inference: shipped with the October 2025 plugins beta.
3. **The original "Not Planned" rationale for #19276.** Label was `stale` (automated). Whether Anthropic explicitly decided against support or it lapsed and was implemented quietly post-plugins-beta is unclear.
4. **Plan-mode propagation to subagents (#43777).** Open as of search time. No release note has addressed it.
5. **`InstructionsLoaded` first-shipped version.** Documented in the current reference; no release note bullet announces it as a new event. Community blog dates suggest late 2025 / early 2026 alongside `ConfigChange` and `CwdChanged`.
6. **`WorktreeCreate` / `WorktreeRemove` first-shipped version.** Same as above — present in the current reference, never given a launch bullet. Probably shipped with worktree-isolation infrastructure in late 2025.
7. **Whether `EnterPlanMode` is *intentionally* stripped from subagents.** Community behavior says yes; no Anthropic doc confirms design rather than bug.
8. **Plugin-shipped agent `permissionMode` restriction history.** Reference says it's "not supported for plugin-shipped agents" — when this restriction was introduced is not in any changelog entry located.

---

## 16. Verbatim Quotes Library (slide-ready)

Each H3 below holds one verbatim quote ready to drop into a presentation slide, with full attribution: post title, date, URL, and the architectural principle it supports.

### "Plugins package any combination of slash commands, subagents, MCP servers, and hooks"

> "Plugins are a lightweight way to package and share any combination of: Slash commands, Subagents, MCP servers, and Hooks."

— **Customize Claude Code with plugins** (Anthropic, 2025-10-09) — https://www.anthropic.com/news/claude-code-plugins (canonical: https://claude.com/blog/claude-code-plugins). Supports Section 2 "Plugin = security perimeter" and the launch-day framing.

### "Hooks are deterministic and guarantee the action happens"

> "Hooks run scripts automatically at specific points in Claude's workflow. Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens."

— **Best practices for Claude Code** (Anthropic, continuously updated) — https://code.claude.com/docs/en/best-practices. Supports Section 4 "PreToolUse Bash allowlist hook" and the broader belt-and-suspenders argument.

### "Find the smallest possible set of high-signal tokens"

> "Good context engineering means finding the smallest possible set of high-signal tokens that maximize desired outcome likelihood."

— **Effective context engineering for AI agents** (Anthropic Engineering, 2025-09-29) — https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents. The single most quotable line for a plugin-author talk — it justifies skills, lazy loading, and the subagent design space.

### "Progressive disclosure is the core design principle"

> "Progressive disclosure is the core design principle that makes Agent Skills flexible and scalable."

— **Equipping agents for the real world with Agent Skills** (Anthropic Engineering, 2025-10-16) — https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills. Supports the read-only/mutate-state split in skills (Section 2 "Bundle the dependencies").

### "Simple, composable patterns"

> "The most successful implementations weren't using complex frameworks or specialized libraries. Instead, they were building with simple, composable patterns."

— **Building effective agents** (Anthropic Engineering, 2024-12-19) — https://www.anthropic.com/engineering/building-effective-agents. The single best argument for "don't bake everything into a mega-agent" (Section 2).

### "Gather context, take action, verify work, repeat"

> "gather context → take action → verify work → repeat"

— **Building agents with the Claude Agent SDK** (Anthropic, 2025-09-29) — https://claude.com/blog/building-agents-with-the-claude-agent-sdk. Maps directly onto trust-but-verify after every fan-out (Section 11).

### "Multi-agent systems spend more tokens to solve harder problems"

> "Multi-agent systems work mainly because they help spend enough tokens to solve the problem."

— **How we built our multi-agent research system** (Anthropic Engineering, 2025-06-13) — https://www.anthropic.com/engineering/multi-agent-research-system. Informs the user-interaction budget (Section 9): if you're spending more tokens than a single agent, make sure the deliverable is demonstrably bigger.

### "Tools are a contract between deterministic systems and non-deterministic agents"

> "Tools are a new kind of software which reflects a contract between deterministic systems and non-deterministic agents."

— **Writing effective tools for agents — with agents** (Anthropic Engineering, 2025-09-11) — https://www.anthropic.com/engineering/writing-tools-for-agents. Supports Section 6 "Manifest as contract" and the explicit-permission-contracts theme.

### "Code execution with MCP: 150,000 → 2,000 tokens"

> "This reduces the token usage from 150,000 tokens to 2,000 tokens — a time and cost saving of 98.7%."

— **Code execution with MCP: building more efficient agents** (Anthropic Engineering, 2025-11-04) — https://www.anthropic.com/engineering/code-execution-with-mcp. Supports the "small composable tool calls" principle (Section 5 atomic-Bash) and the case for filesystem-of-tools over context-bloating MCP.

### "Harnesses encode assumptions that go stale as models improve"

> "Harnesses encode assumptions that go stale as models improve."

— **Scaling Managed Agents: Decoupling the brain from the hands** (Anthropic Engineering, 2026-04-08) — https://www.anthropic.com/engineering/managed-agents. The case for fresh-install testing (Section 10): your harness assumptions may already be stale.

### "Past every code review and every test"

> "The changes it introduced made it past multiple human and automated code reviews, as well as unit tests, end-to-end tests, automated verification, and dogfooding."

— **An update on recent Claude Code quality reports (April 23 postmortem)** (Anthropic Engineering, 2026-04-23) — https://www.anthropic.com/engineering/april-23-postmortem. The strongest argument for plugin-side end-to-end smoke tests (Section 10): if Anthropic's pipeline can ship a regression, yours definitely needs more verification than "the dev runs work."

### "Any git repo with a properly formatted marketplace.json"

> "To host a marketplace, all you need is a git repository, GitHub repository, or URL with a properly formatted `.claude-plugin/marketplace.json` file."

— **Customize Claude Code with plugins** (Anthropic, 2025-10-09) — https://claude.com/blog/claude-code-plugins. The two-repo distribution model (Section 2) — marketplaces are intentionally lightweight.

### "Subagents enable parallelization and manage context"

> "Subagents are useful for two main reasons. First, they enable parallelization… Second, they help manage context."

— **Building agents with the Claude Agent SDK** (Anthropic, 2025-09-29) — https://claude.com/blog/building-agents-with-the-claude-agent-sdk. Validates the orchestrator-plus-specialists pattern (Section 2 / Section 8).

### "Skills extend Claude by packaging your expertise"

> "Skills extend Claude's capabilities by packaging your expertise into composable resources for Claude, transforming general-purpose agents into specialized agents that fit your needs."

— **Equipping agents for the real world with Agent Skills** (Anthropic Engineering, 2025-10-16) — https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills. Supports the bundling-as-distribution model (Section 2 "Bundle the dependencies you can").

### "Effective sandboxing requires both filesystem and network isolation"

> "Effective sandboxing requires both filesystem and network isolation. Without network isolation, a compromised agent could exfiltrate sensitive files like SSH keys."

— **Beyond permission prompts: making Claude Code more secure and autonomous** (Anthropic Engineering, 2025-10-20) — https://www.anthropic.com/engineering/claude-code-sandboxing. Supports Section 6 "Plugin = security perimeter" — security in agentic systems requires layered isolation, not just rule-based denial.

### "Auto mode is not a drop-in replacement for human review"

> "It is not a drop-in replacement for careful human review on high-stakes infrastructure."

— **Claude Code auto mode: a safer way to skip permissions** (Anthropic Engineering, 2026-03-25) — https://www.anthropic.com/engineering/claude-code-auto-mode. Important caveat when discussing the PreToolUse allowlist pattern (Section 4) — auto mode and plugin-level allowlists serve different threat models.

### "Write extremely high-quality tests"

> "Write extremely high-quality tests… Claude will work autonomously to solve whatever problem I give it. So it's important that the task verifier is nearly perfect, otherwise Claude will solve the wrong problem."

— **Building a C compiler with a team of parallel Claudes** (Anthropic Engineering, 2026-02-05) — https://www.anthropic.com/engineering/building-c-compiler. Supports both Section 10 (fresh-install verification) and Section 11 (trust-but-verify). The verifier is the load-bearing component, not the prompt.

### "Compaction isn't sufficient for production-quality apps"

> "Compaction isn't sufficient. Out of the box, even a frontier coding model like Opus 4.5… will fall short of building a production-quality web app."

— **Effective harnesses for long-running agents** (Anthropic Engineering, 2025-11-26) — https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents. Supports the case for structured envelopes and explicit handoffs (Section 11 Mitigation 2 / Section 9 three-step parent-owned pattern).

### "Find the simplest solution and increase complexity only when needed"

> "Find the simplest solution possible, and only increase complexity when needed."

— **Harness design for long-running application development** (Anthropic Engineering, 2026-03-24) — https://www.anthropic.com/engineering/harness-design-long-running-apps. The architectural meta-principle behind Section 14 Theme 1 (explicit permission contracts) and Theme 6 (match the tool's grain).

### "Separate the agent doing the work from the agent judging it"

> "Separating the agent doing the work from the agent judging it proves to be a strong lever to address… self-evaluation issues."

— **Harness design for long-running application development** (Anthropic Engineering, 2026-03-24) — https://www.anthropic.com/engineering/harness-design-long-running-apps. Supports the orchestrator-as-verifier pattern (Section 8 trust-but-verify) and the parent-owned-Q&A pattern (Section 9).

### "Subagents inherit only what the parent passes in the prompt"

> "Sub-agents run in isolated context — they cannot see the parent's history except what's passed in the prompt."

— Video: **Claude Code Sub-Agents: Step-by-Step Beginner Tutorial** (2026-03-31) — https://www.youtube.com/watch?v=MbKUeufUZIY. Validates the structured-envelope contract (Section 11 Mitigation 2) — the only way subagents communicate is via inputs and outputs you design.

### "Always exit with code 0 from a hook"

> "Always exit with code 0 (success) so we don't interrupt Claude's work, even if something went wrong."

— Greg Baugues — Claude Code: Getting Started with Hooks (2025-07-10) — https://www.baugues.com/hooks/ (companion blog to video https://www.youtube.com/watch?v=8T0kFSseB58). Defensive-default for hook scripts; supports Section 11 observability layer.

### "Hooks are the enforcement layer that subagents cannot bypass" *(paraphrase)*

> "Hooks are the only place to enforce policy that a sub-agent cannot bypass." *(paraphrase — see attribution note)*

— IndyDevDan video: **I'm HOOKED on Claude Code Hooks: Advanced Agentic Coding** — https://www.youtube.com/watch?v=J5B9UGTuNoM. **Note:** this entry is a paraphrase — the transcript was not fetched, but the framing is consistent across multiple community references in our notes. Every other entry in this Section is verbatim with attribution. The canonical framing for plugin-level PreToolUse hooks (Section 4) and the security-perimeter theme (Section 6).

---

## 17. Community Video References

The expert-videos research surveyed roughly 35 YouTube videos plus 6 long-form blog/podcast companions. This section is the curated subset most useful for plugin authors: the talk-prep top 10, the cross-reference table to our findings, and the channels worth subscribing to.

### Top 10 videos for talk prep

Ranked by utility against a conference talk validating plugin best-practices.

1. **Claude Agent SDK [Full Workshop] — Thariq Shihipar (Anthropic, 2026-01-05)** — https://www.youtube.com/watch?v=TqC1qOfiVcQ — the authoritative Anthropic-engineer walkthrough of the SDK that powers Claude Code; ground-truth for any plugin claim about how the internal loop works.
2. **I'm HOOKED on Claude Code Hooks: Advanced Agentic Coding — IndyDevDan** — https://www.youtube.com/watch?v=J5B9UGTuNoM — the canonical "hooks as defense-in-depth + observability" story; matches our `block-installs.py` and atomic-Bash patterns.
3. **#412 Max: The Boris Cherny Workflow (Mastering Claude Code 2026)** — https://www.youtube.com/watch?v=QZCJ4aEhrMk — Boris Cherny's setup as authoritative counter-anchor (vanilla, plan-mode-first, parallel worktrees, team CLAUDE.md); the "plugin authors should augment, not replace" framing benefits from quoting him directly.
4. **Advanced Claude Code techniques — John Lindquist on Lenny's Podcast (2026-01-26)** — https://www.lennysnewsletter.com/p/advanced-claude-code-techniques-context — long-form, advanced, transcript-rich; concrete patterns (stop hooks for TS validation, mermaid diagrams as context) you can lift directly.
5. **Claude 4 ADVANCED AI Coding: Parallelize Claude Code with Git Worktrees — IndyDevDan (2025-05-26)** — https://www.youtube.com/watch?v=f8RnRuaxee8 — the origin story of the worktrees-per-agent pattern that became native; good chronology slide.
6. **Claude Code Sub-Agents: Step-by-Step Beginner Tutorial (2026-03-31)** — https://www.youtube.com/watch?v=MbKUeufUZIY — clean primer to send audience members to before/after the talk.
7. **How To Use Claude Code Sub-Agents Better Than 99% of People (2026-04-01)** — https://www.youtube.com/watch?v=6mG6tS6WG00 — the six-pattern taxonomy (orchestrator, fan-out, validation chain, specialist routing, hierarchy, watchdog) is excellent slide material.
8. **Anthropic's Full Claude Skills Guide In 22 Minutes (2026-02-15)** — https://www.youtube.com/watch?v=TzJecWCbex0 — compressed authoritative skills intro; cite for "description is a routing signal, not documentation."
9. **Agent Skills: Code Beats Markdown (Here's Why) — Sam Witteveen** — https://www.youtube.com/watch?v=IjiaCOt7bP8 — concise counter-take justifying our `scripts/` + `SKILL.md` split (scripts for transformation, markdown for judgment).
10. **Code with Claude 2026: Opening Keynote — Anthropic (2026-05-06)** — https://www.youtube.com/watch?v=GMIWm5y90xA — the platform-direction anchor: Managed Agents, Proactive Workflows, official plugin directory; use as the "where this is going" closer.

Honorable mention: **Greg Baugues — Getting Started with Hooks** — https://www.youtube.com/watch?v=8T0kFSseB58 — the only video whose companion blog (https://www.baugues.com/hooks/) was extractable in our research; useful as the "smallest working hook example" slide.

### Cross-references to our findings

Mapping shipped-plugin findings to community videos that discuss the same problem or pattern. "No direct video coverage" means original to our work.

| Our Finding | Video Coverage | Notes |
|---|---|---|
| **DBT F1** — Plugin = distribution unit, not "loose skill" | AICodeKing — "Top 10 Skills, Plugins & CLIs (April 2026)" (https://www.youtube.com/watch?v=KjEFy5wjFQg); Anthropic plugin-directory launch coverage | Community treats bundling as default once you have >2 sub-components. |
| **DBT F2** — Verb-led skill descriptions for discovery | "Anthropic's Full Claude Skills Guide In 22 Minutes" (https://www.youtube.com/watch?v=TzJecWCbex0); "Claude Code's Creator Uses These Skills" (https://www.youtube.com/watch?v=AhXfI1rSUPc) | Both videos explicitly call out the description-as-routing-signal pattern. |
| **DBT F3** — Sub-agents with narrow tool allowlists | "Claude Code Sub-Agents: Step-by-Step Beginner Tutorial" (https://www.youtube.com/watch?v=MbKUeufUZIY); "How To Use Sub-Agents Better Than 99%" (https://www.youtube.com/watch?v=6mG6tS6WG00) | Least-privilege tool sets are mainstream community practice. |
| **DBT F4** — Structured handoffs between phases | "How To Use Sub-Agents Better Than 99% of People" (validation-chain pattern) | Same shape as our staging → mart → test pipeline. |
| **DBT F5** — Plan-mode-first development | "Even Anthropic Engineers Use This Claude Code Workflow" (https://www.youtube.com/watch?v=ASAaKhK1B5w); "#412 Boris Cherny Workflow"; "How I use Claude Code for real engineering" (https://www.youtube.com/watch?v=kZ-zzHVUrO4) | This is *the* most universally-confirmed pattern across creators. |
| **DBT F6** — CLAUDE.md as living team style guide | "#412 Boris Cherny Workflow"; "Even Anthropic Engineers..."; John Lindquist on Lenny's Podcast | Boris Cherny's team-shared CLAUDE.md committed to git is the canonical version. |
| **DBT F7 / F9** — Atomic Bash commands (no compound shell) | **No direct video coverage** — original lesson. | Our atomic-Bash rule is *not* discussed in any video we surveyed. Worth a talk slot — community blind spot. |
| **DBT F8** — Spawn isolated workspaces, not just isolated contexts | "Parallelize Claude Code with Git Worktrees — IndyDevDan" (https://www.youtube.com/watch?v=f8RnRuaxee8); "Parallel Claude Code + Git Worktrees" (https://www.youtube.com/watch?v=rFGlJ4oIlhw) | Worktrees are the canonical isolation primitive. |
| **DBT F9** — Hooks as defense-in-depth + observability | "I'm HOOKED on Claude Code Hooks — IndyDevDan" (https://www.youtube.com/watch?v=J5B9UGTuNoM); Greg Baugues "Getting Started with Hooks" (https://www.youtube.com/watch?v=8T0kFSseB58); Lukasz Fryc's "20+ Examples" (https://dev.to/lukaszfryc/claude-code-hooks-complete-guide-with-20-ready-to-use-examples-2026-dcg) | Universally recognized; our `block-installs.py` matches `claude-code-damage-control`. |
| **DBT F10** — Skills for judgment, scripts for transformation | "Agent Skills: Code Beats Markdown — Sam Witteveen" (https://www.youtube.com/watch?v=IjiaCOt7bP8); Thariq Shihipar on HTML-as-spec (https://www.youtube.com/watch?v=Qrpm7E80wQ0) | Sam Witteveen's video is the cleanest articulation of this split. |
| **Fabric N1** — Notebook templates as skill resources | No direct video coverage — original lesson. | Notebook-as-skill-template is novel to our Fabric work. |
| **Fabric N2** — Bronze/silver/gold layering inside one plugin | No direct video coverage — original lesson. | Medallion-architecture-aware plugins are not yet a community topic. |
| **Fabric N3** — Fabric CLI wrapping inside a skill | No direct video coverage — original lesson. | |
| **Fabric N4** — Preflight checks before long migrations | "The Ultimate Claude Code Guide \| MCP, Skills & More" (https://www.youtube.com/watch?v=uogzSxOw4LU) (briefly: MCP preflight) | General preflight principle is community-known; Fabric-specific preflight is original. |
| **Fabric N5** — Lakehouse SQL endpoint as read-only validator | No direct video coverage — original lesson. | |
| **Fabric N6** — M-to-PySpark code translation as skill | No direct video coverage — original lesson. | Cross-language code translation skills are rare in surveyed content. |
| **Fabric N7** — Parallel notebook generation via sub-agents | "Claude Code's Agent Teams Are Insane" (https://www.youtube.com/watch?v=-1K_ZWDKpU0); "How To Use Sub-Agents Better Than 99%" | Fan-out pattern is community-validated; Fabric-specific use is original. |
| **Fabric N8** — Trust-but-verify after every fan-out | Implicit in worktrees-parallel videos but not articulated as a verifier requirement. | Auto-reporting unknown patterns back to plugin authors is novel. |
| **Fabric N9 / N10 / N11** — Lakehouse/Warehouse, Entra ID auth, notebook deployer batching | No direct video coverage — original lessons. | |
| **Fabric N12** — Project-config.yml as plugin discovery anchor | No direct video coverage — original lesson. | Several videos mention `.claude/` discovery, none discuss project-level YAML config. |
| **Fabric N13** — UTF-8 BOM for PowerShell | No direct video coverage — original lesson. | |
| **Fabric N14** — Background subagents cannot read plugin-cache | No direct video coverage — original lesson. | The convention (numbered folders) is project-specific; the principle (predictable output paths) is community-known. |
| **Fabric N15** — `AskUserQuestion` not in any subagent | No direct video coverage. | The "skill = transformation" / "skill = judgment" split leaves room for "skill = read-only adapter" as a third category. |
| **Fabric N16 / N17** — Ancestor walk + sentinel; builder pollution | No direct video coverage — original lessons. | Closest: John Lindquist on "documentation that serves humans + machines." |

**Summary:** roughly 9/10 DBT findings are community-validated to some degree; roughly 2/17 Fabric findings have meaningful community parallel. The Fabric work is largely **original territory** — that's a talk angle in itself: "here's what the community knows, here's the gap I closed."

### Channels worth subscribing to

Ranked by signal-to-noise for plugin / agent / hook / skill work specifically.

1. **IndyDevDan (@indydevdan)** — https://www.youtube.com/@indydevdan — highest density of substantive plugin/hooks/sub-agents content; the canonical hooks educator. Pair with his GitHub (https://github.com/disler) for runnable reference implementations (`claude-code-hooks-mastery`, `claude-code-hooks-multi-agent-observability`, `claude-code-hooks-damage-control`).
2. **AnthropicAI (@AnthropicAI)** — https://www.youtube.com/@AnthropicAI — first-party source for keynotes, official skill / plugin launches, Boris Cherny / Thariq Shihipar talks. Lower volume but ground-truth.
3. **Sam Witteveen (@samwitteveenai)** — https://www.youtube.com/@samwitteveenai — strong on agent frameworks, MCP fundamentals, and contrarian takes (code-beats-markdown). Long-running channel — historical context for how the field got here.
4. **Greg Baugues (@gregcode)** — https://www.youtube.com/@gregcode — practical, single-file, runnable hook examples. Best "show me the smallest working version" channel.
5. **Cole Medin (@ColeMedin)** — https://www.youtube.com/@ColeMedin — high-tempo (twice-weekly), 175k+ subscribers, PIV-loop methodology, agentic engineering focus. Pair with https://github.com/coleam00/ai-transformation-workshop.
6. **John Lindquist (egghead.io)** — workshops at https://egghead.io/workshop/claude-code; appearances on Lenny's Podcast and others. Less YouTube-native but exceptionally substantive when he does appear.
7. **AI Jason** — frequent Claude Code multi-agent walkthroughs; reasonable signal but more breadth than depth.
8. **AICodeKing** — survey/roundup channel; useful for "what's new this month" framing but lower per-video depth.
9. **Matthew Berman (@matthew_berman)** — broad-AI channel; Claude Code coverage is news-cycle-driven rather than deep-dive — useful for dating major releases.
10. **Patrick Loeber (@patloeber)** — pivoted to Google DeepMind dev rel; less Claude-Code-specific now, but historical content on Python/agents/SDK patterns is worth searching by topic.

**Submarine channels** (cited by the listed channels, less direct content but worth following): Pragmatic Engineer (Gergely Orosz, written interviews with Boris Cherny — https://newsletter.pragmaticengineer.com/), Lenny's Podcast (long-form interviews including John Lindquist), Class Central (curated talk indexes).

### Reference implementations on GitHub

Worth bookmarking alongside the videos:

- **disler/claude-code-hooks-mastery** — canonical reference for naming/structuring hooks.
- **disler/claude-code-hooks-multi-agent-observability** — hook-events-to-websocket-stream pattern for live monitoring (basis for our `agent-logger.py`).
- **disler/claude-code-hooks-damage-control** — formalizes defense-in-depth via PreToolUse blocking of `rm -rf` and sensitive-file edits (parallel to our `block-installs.py`).
- **anthropics/claude-plugins-official** — Anthropic-managed directory of high-quality Claude Code plugins (launched 2026-05-22) — https://github.com/anthropics/claude-plugins-official.
- **anthropics/skills** — Anthropic's official skills repo; canonical reference layout.

---

## 18. Anthropic Engineering Posts: The Canon

The full bookmarkable list of Anthropic blog posts that inform plugin best practices, organized by theme. Every link is a primary source.

### Plugins & extensibility

- **Customize Claude Code with plugins** (2025-10-09) — https://www.anthropic.com/news/claude-code-plugins / canonical https://claude.com/blog/claude-code-plugins
- **Best practices for Claude Code** (continuously updated) — https://code.claude.com/docs/en/best-practices
- **Plugins reference** (canonical schema spec) — https://code.claude.com/docs/en/plugins-reference
- **Hooks reference** — https://code.claude.com/docs/en/hooks
- **Sub-agents reference** — https://code.claude.com/docs/en/sub-agents
- **Official plugin directory** (launched 2026-05-22) — https://github.com/anthropics/claude-plugins-official

### Agent SDK & multi-agent orchestration

- **Building agents with the Claude Agent SDK** (2025-09-29) — https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk / canonical https://claude.com/blog/building-agents-with-the-claude-agent-sdk
- **Building effective agents** (2024-12-19) — https://www.anthropic.com/engineering/building-effective-agents — the foundational "workflows vs agents" essay.
- **How we built our multi-agent research system** (2025-06-13) — https://www.anthropic.com/engineering/multi-agent-research-system
- **Building a C compiler with a team of parallel Claudes** (2026-02-05) — https://www.anthropic.com/engineering/building-c-compiler
- **Effective harnesses for long-running agents** (2025-11-26) — https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- **Harness design for long-running application development** (2026-03-24) — https://www.anthropic.com/engineering/harness-design-long-running-apps
- **Scaling Managed Agents: Decoupling the brain from the hands** (2026-04-08) — https://www.anthropic.com/engineering/managed-agents
- **Enabling Claude Code to work more autonomously** (2025-09-29) — https://www.anthropic.com/news/enabling-claude-code-to-work-more-autonomously — the "Hooks + Subagents + Background tasks + Checkpoints" autonomy bundle.
- **Demystifying evals for AI agents** (2026-01-09) — https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents

### Skills

- **Introducing Agent Skills** (2025-10-16) — https://www.anthropic.com/news/skills / canonical https://claude.com/blog/skills
- **Equipping agents for the real world with Agent Skills** (2025-10-16) — https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- **Agents for Financial Services** (2026-05-05) — https://www.anthropic.com/news/finance-agents — confirms "skills + connectors + subagents" as canonical packaging unit.

### MCP

- **Introducing the Model Context Protocol** (2024-11-25) — https://www.anthropic.com/news/model-context-protocol
- **Claude can now connect to your world (Integrations / remote MCP)** (2025-05-01) — https://www.anthropic.com/news/integrations / canonical https://claude.com/blog/integrations
- **Code execution with MCP: building more efficient agents** (2025-11-04) — https://www.anthropic.com/engineering/code-execution-with-mcp
- **Donating MCP to the Agentic AI Foundation** (2025-12-09) — https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation
- **New capabilities for building agents on the Anthropic API** (2025-05-22) — https://www.anthropic.com/news/agent-capabilities-api / canonical https://claude.com/blog/agent-capabilities-api
- **Introducing advanced tool use on the Claude Developer Platform** (2025-11-24) — https://www.anthropic.com/engineering/advanced-tool-use

### Prompt & context engineering

- **Effective context engineering for AI agents** (2025-09-29) — https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- **Writing effective tools for agents — with agents** (2025-09-11) — https://www.anthropic.com/engineering/writing-tools-for-agents
- **The "think" tool: enabling Claude to stop and think in complex tool use situations** (2025-03-20) — https://www.anthropic.com/engineering/claude-think-tool
- **Prompt caching with Claude** (2024-08-14, GA update 2024-12-17) — https://www.anthropic.com/news/prompt-caching / canonical https://claude.com/blog/prompt-caching

### Safety, permissions, postmortems

- **Beyond permission prompts: making Claude Code more secure and autonomous** (2025-10-20) — https://www.anthropic.com/engineering/claude-code-sandboxing
- **Claude Code auto mode: a safer way to skip permissions** (2026-03-25) — https://www.anthropic.com/engineering/claude-code-auto-mode
- **An update on recent Claude Code quality reports (April 23 postmortem)** (2026-04-23) — https://www.anthropic.com/engineering/april-23-postmortem

### Models (latest five; useful for "what Claude can do today" framing)

- **Claude Opus 4.7** (2026-04-16) — https://www.anthropic.com/news/claude-opus-4-7
- **Claude Sonnet 4.6** (2026-02-17) — https://www.anthropic.com/news/claude-sonnet-4-6
- **Claude Opus 4.6** (2026-02-05) — https://www.anthropic.com/news/claude-opus-4-6
- **Claude Opus 4.5** (2025-11-24) — https://www.anthropic.com/news/claude-opus-4-5
- **Claude Haiku 4.5** (2025-10-15) — https://www.anthropic.com/news/claude-haiku-4-5 — the "fast subagent" model: *"Sonnet 4.5 can break down a complex problem into multi-step plans, then orchestrate a team of multiple Haiku 4.5s to complete subtasks in parallel."*
- **Claude Sonnet 4.5** (2025-09-29) — https://www.anthropic.com/news/claude-sonnet-4-5 — co-shipped with the Claude Agent SDK rebrand and the autonomy update.

### Primary-source changelog / release notes

- **Claude Code CHANGELOG (raw, ~320 KB)** — https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md
- **Claude Platform / API release notes** — https://platform.claude.com/docs/en/release-notes/api

### Critical GitHub issues for plugin authors

- **#19276** — Custom Subagent Support in Task Tool (Not Planned) — https://github.com/anthropics/claude-code/issues/19276
- **#27145** — `CLAUDE_PLUGIN_ROOT` not set for SessionStart hooks — https://github.com/anthropics/claude-code/issues/27145
- **#36585** — `CLAUDE_PLUGIN_ROOT` not passed to UserPromptSubmit hooks — https://github.com/anthropics/claude-code/issues/36585
- **#38699** — `CLAUDE_PLUGIN_ROOT` inconsistent between hooks/skills — https://github.com/anthropics/claude-code/issues/38699
- **#18721** — Missing warning for AskUserQuestion subagent limitation — https://github.com/anthropics/claude-code/issues/18721
- **#34592** — AskUserQuestion unavailable in all sub-agent contexts — https://github.com/anthropics/claude-code/issues/34592
- **#43777** — Plan mode constraint not propagated to Agent tool subagents — https://github.com/anthropics/claude-code/issues/43777
- **#24072** — Skill tool not available in Plan mode — https://github.com/anthropics/claude-code/issues/24072
- **#39455** — Plugin userConfig values not prompted on enable — https://github.com/anthropics/claude-code/issues/39455

*(Issue #32731 — "Teammates have fewer tools than subagents" — is also tracked, but is not cited inline anywhere in Sections 15 or below. Kept out of this canon to preserve the "every entry is woven into the body" property; track upstream if it becomes relevant to a finding.)*

---

## Appendix: Where to read more

### Companion best-practices docs
- `how-to-create-plugins.md` — step-by-step build/release mechanics
- `claude-code-agent-best-practices.md` — agent patterns; "Agent Patterns from Shipped Plugins (2026)" section cross-references findings here
- `claude-code-skill-best-practices.md` — skill patterns; "Skill Patterns from Shipped Plugins (2026)" section is the skill counterpart
- `claude-code-hooks-best-practices.md` — hook patterns; PreToolUse auto-approval pattern in depth

### Plugin source receipts
- `dbt-pipeline-toolkit/_Documentation/plugin_learnings.md` — DBT plugin's 10 findings (F1–F10) plus the talk themes section
- `fabric-dataflow-migration-toolkit/_Documentation/plugin_learnings.md` — Fabric plugin's 17 findings (N1–N17) plus the inherited dbt-findings section

### External citations

All primary-source citations (changelog quotes, Anthropic blog quotes, community video references, and GitHub issue links) are now embedded inline in Sections 15–18 of this document. The earlier `claude-code-changelog-research-2026.md`, `anthropic-official-blog-research-2026.md`, and `claude-code-expert-videos-research-2026.md` research files have been consolidated into this single authoritative reference.

Every section in this doc cross-references findings from the source `plugin_learnings.md` docs. Section 15 cross-references those findings to official Anthropic sources. Section 16 is the slide-ready quote library. Section 17 is the community video index. Section 18 is the Anthropic engineering canon. Dig into the source `plugin_learnings.md` docs for the long-form narrative; this doc is the distilled best-practices view with all external receipts attached.
