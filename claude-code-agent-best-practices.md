# Claude Code Agent Best Practices

**Last Updated:** 2026-05-25
**Sources:** Anthropic official documentation, Claude Code release notes (v2.1.111–v2.1.150), Anthropic engineering blog (22 primary-source posts), expert YouTube creators, plus hard-won lessons from two production plugins (`dbt-pipeline-toolkit`, `fabric-dataflow-migration-toolkit`).

> **Companion documents:**
> - `how-to-create-agents.md` — step-by-step build guide
> - `how-to-create-skills.md` — step-by-step skill guide
> - `how-to-create-hooks.md` — step-by-step hook guide
> - `how-to-create-plugins.md` — step-by-step plugin guide
> - `claude-code-skill-best-practices.md` — skill-level patterns
> - `claude-code-hooks-best-practices.md` — hook patterns
> - `claude-code-plugin-best-practices.md` — plugin patterns

---

## 1. Agent Architecture Fundamentals

### 1.1 Single-responsibility principle

Each agent should have **one clear responsibility**, not broad capabilities. The symptom of an over-broad agent is a description with "and" joining multiple responsibilities.

Two orchestration shapes dominate:

**(a) Main agent as orchestrator** (most common) — the main Claude Code session plans, delegates, and synthesizes. Best for single-session work.

**(b) Dedicated orchestrator agent** (run via `claude --agent orchestrator`) — the main thread is itself a specialized orchestrator subagent whose only job is delegating. Best for repeated workflows.

Critical constraint: **subagents cannot spawn subagents** — Claude Code enforces a one-level hierarchy at the harness level. Use chains, parent-driven loops, or Agent Teams for hierarchical work. (See §6.4 for how this constraint shapes plugin orchestrators.)

> "Subagents are useful for two main reasons. First, they enable parallelization… Second, they help manage context." — *Building agents with the Claude Agent SDK* (2025-09-29)

### 1.2 Context window management

- Claude supports **200K–1M tokens** depending on model (Opus 4.7 / Sonnet 4.6 1M context in beta).
- **Don't fill the entire context window** — performance degrades as it fills. Per Anthropic best-practices: *"Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills."*
- Compress global state aggressively — store only plan, key decisions, latest artifacts.
- Prefer retrieval and summaries over dumping raw logs.
- Subagents auto-compact at ~95% capacity (override with `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`).
- Aim for **250–500 lines per agent file** — not 2K+.

> "Good context engineering means finding the smallest possible set of high-signal tokens that maximize desired outcome likelihood." — *Effective context engineering for AI agents* (2025-09-29)

### 1.3 CLAUDE.md hierarchy

| Level | Path | Loaded |
|---|---|---|
| Global | `~/.claude/CLAUDE.md` | Every session, every project |
| Project root | `CLAUDE.md` or `.claude/CLAUDE.md` | Sessions opened inside the project |
| Folder | `src/CLAUDE.md` | Lazily when files in that folder are touched |
| `--add-dir` dirs | (not loaded by default) | Set `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` |

Teammates load CLAUDE.md from their CWD. Per Boris Cherny's workflow (Pragmatic Engineer interview): a single team-shared CLAUDE.md, committed to git, updated multiple times per week when Claude does something wrong, is *the* leverage point — invest there before agents.

> "CLAUDE.md is loaded every session, so only include things that apply broadly. For domain knowledge or workflows that are only relevant sometimes, use skills instead." — *Best practices for Claude Code*

### 1.4 The agent loop

The canonical loop, codified by the Claude Agent SDK team:

> **"gather context → take action → verify work → repeat"** — *Building agents with the Claude Agent SDK*

This shape applies recursively at every level — single tool call, subagent invocation, full pipeline. The "verify" step is the highest-leverage one and is what most failing agents skip.

> "Give Claude a way to verify its work. Include tests, screenshots, or expected outputs so Claude can check itself. This is the single highest-leverage thing you can do." — *Best practices for Claude Code*

---

## 2. Agent Description Writing (Critical for Auto-Delegation)

Claude selects subagents by reading `description` fields. Strong descriptions dramatically improve auto-delegation accuracy. From the *Anthropic's Full Claude Skills Guide* (2026-02-15): *the description line is not documentation — it is a routing signal*.

### Template
> "[What it does]. [Deliverable]. Use [proactively / immediately after X / MUST BE USED for Y]."

### Trigger phrases that work
- "use proactively"
- "MUST BE USED when..."
- "Use immediately after..."
- "Use after spec exists; produce ADR and guardrails"

### Examples
- Good — "Expert code review specialist. Proactively reviews code for quality, security, and maintainability. Use immediately after writing or modifying code."
- Good — "Debugging specialist for errors, test failures, and unexpected behavior. Use proactively when encountering any issues."
- Bad — "General helper for code stuff." (too vague — symptom of over-broad agent)

Other rules:
- **Action-oriented** — agents match on what they *produce*.
- **Explicit deliverable** in the description sentence.
- **Lowercase-hyphen slugs** traceable across queue entries, branches, commit messages.

---

## 3. Frontmatter Reference (2026)

```yaml
---
name: code-reviewer                    # Required: lowercase-hyphen
description: Reviews code changes...   # Required: guides auto-delegation
tools: Read, Grep, Glob, Bash          # Optional allowlist (inherits all if omitted)
disallowedTools: Write, Edit           # Optional denylist
model: sonnet                          # sonnet|opus|haiku|claude-opus-4-7|inherit
permissionMode: acceptEdits            # default|acceptEdits|auto|dontAsk|bypassPermissions|plan
maxTurns: 20                           # Cap on agentic turns
skills:                                # Skills preloaded at startup (2-part namespace for plugin skills)
  - my-plugin:api-conventions
mcpServers:                            # MCP servers scoped to this agent
  - github
hooks:                                 # Lifecycle hooks scoped to this agent
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate.sh"
memory: project                        # user|project|local — persistent memory dir
background: true                       # Always run as background task
effort: high                           # low|medium|high|max|xhigh (xhigh added v2.1.111)
isolation: worktree                    # Run in temp git worktree (only valid value)
color: blue                            # red|blue|green|yellow|purple|orange|pink|cyan
initialPrompt: "Start by reading..."   # Auto-submitted first turn when run as --agent main session
---
```

### What plugin-shipped agents support

Per the canonical [plugins reference](https://code.claude.com/docs/en/plugins-reference):

> "Plugin-shipped agents support frontmatter fields: `name`, `description`, `model`, `effort`, `maxTurns`, `tools`, `disallowedTools`, `skills`, `memory`, `background`, and `isolation`. The only valid `isolation` value is `'worktree'`. **For security reasons, `hooks`, `mcpServers`, and `permissionMode` are not supported for plugin-shipped agents.**"

The stripping is intentional, not a bug. Push hooks and MCP servers up to `plugin.json`; push permission decisions to the call site. (See §6.2.)

### Model resolution order
1. `CLAUDE_CODE_SUBAGENT_MODEL` env var
2. Per-invocation `model` parameter passed by parent
3. Subagent definition's `model` frontmatter
4. Main conversation's model

Default is `inherit`.

### Invocation patterns
1. **Automatic delegation** — Claude reads `description` fields and picks
2. **Natural language** — "Use the test-runner subagent to fix failing tests"
3. **@-mention** — `@"code-reviewer (agent)" look at the auth changes` (guarantees that agent)
4. **Session-wide via `--agent`** — replaces main system prompt (`v2.1.119`: `--agent <name>` honors `permissionMode`)
5. **Settings default** — `{"agent": "code-reviewer"}` in `.claude/settings.json`

---

## 4. Tools Configuration

### Least-privilege allowlist

> "Tools are a new kind of software which reflects a contract between deterministic systems and non-deterministic agents." — *Writing effective tools for agents* (2025-09-11)

Always specify `tools:`. Omitting it grants every tool — the **tool sprawl anti-pattern**.

```yaml
tools: Read, Grep, Glob              # Read-only specialist
tools: Read, Edit, Write, Bash       # Implementer
tools: Agent(researcher, planner), Read, TodoWrite  # Orchestrator (delegates only)
```

### Multiple `Agent(...)` types in plugin agents

Before `v2.1.147`, plugin agents declaring multiple `Agent(...)` types in `tools:` silently dropped all but the last entry. *Fixed in v2.1.147:* `"Fixed plugin agents declaring multiple Agent(...) types in tools: frontmatter dropping all but last entry."` If you ship a plugin targeting older versions, list agents in a single comma-separated `Agent(...)` rather than relying on multiple parenthesized entries.

### Subagent `subagent_type` matching

Per `v2.1.140`: *"Improved Agent tool `subagent_type` matching (case/separator insensitive)."* — `Task(subagent_type: "MyAgent")` and `Task(subagent_type: "my-agent")` both resolve correctly on modern versions. Still use canonical lowercase-hyphen names for cross-version compatibility.

### Tools NOT available in subagents

The subagent tool filter strips, in all subagent contexts (foreground or background, in-process or external):

- `AskUserQuestion` — see §6.5
- `EnterPlanMode`, `ExitPlanMode` — see §6.7
- `Task`, `TaskOutput`, `TaskStop`
- `Agent`, `TeamCreate`, `TeamDelete`
- `CronCreate`, `CronDelete`, `CronList`

Confirmed via GitHub Issues #18721, #20275, #34592, #43777, #32731. No public release-note has acknowledged this as intentional design.

### Anthropic on tool design

> "More tools don't always lead to better outcomes. A common error we've observed is tools that merely wrap existing software functionality or API endpoints." — *Writing effective tools for agents*

> "When writing tool descriptions and specs, think of how you would describe your tool to a new hire on your team."

---

## 5. Model Selection

### Model lineage (agent-relevant releases)

| Model | Date | Why it matters for agents |
|---|---|---|
| **Sonnet 4 / Opus 4** | 2025-05-22 | Extended thinking + interleaved thinking; Opus 4 = "sustained performance… several hours" |
| **Sonnet 4.5** | 2025-09-29 | "Best model for building complex agents" — co-shipped with Agent SDK, hooks, subagent delegation, checkpoints |
| **Haiku 4.5** | 2025-10-15 | Near-frontier coding at ~1/3 cost — designed to be orchestrated by Sonnet/Opus for sub-agent fan-out |
| **Opus 4.5** | 2025-11-24 | First model with `effort` parameter; programmatic tool calling + tool search tool launched same day |
| **Opus 4.6 / Sonnet 4.6** | 2026-02-05 / 02-17 | 1M context, compaction API beta. Sonnet 4.6 preferred over Opus 4.5 ~59% of the time. |
| **Opus 4.7** | 2026-04-16 | +13% on 93-task coding benchmark over 4.6; file-system memory; **`xhigh` reasoning tier**; API breaking changes from 4.6 |

### Routing guidance

| Model | Use for |
|---|---|
| **Haiku 4.5** | Explore-style discovery, log processing, data profiling, high-volume read-only work. Quote: *"Sonnet 4.5 can break down a complex problem into multi-step plans, then orchestrate a team of multiple Haiku 4.5s to complete subtasks in parallel."* |
| **Sonnet 4.6** | Code review, implementation, test writing, most work (default). Quote: *"Sonnet 4.6 reasons effectively across all that context. This can make it much better at long-horizon planning."* |
| **Opus 4.7 + `effort: max`/`xhigh`** | Architecture, complex refactors, security audits, "hand off your hardest coding work with confidence" |

### `effort` parameter

GA on Opus 4.6+ (was beta on 4.5). Replaces `budget_tokens`. Values: `low` | `medium` | `high` | `max` | `xhigh` (Opus 4.7 only). Skills can interpolate `${CLAUDE_EFFORT}` (`v2.1.120`). Hooks receive `effort.level` and `$CLAUDE_EFFORT` (`v2.1.133`).

> **Postmortem lesson:** *"On March 4, we changed Claude Code's default reasoning effort from `high` to `medium` to reduce the very long latency… A bug caused this to keep happening every turn for the rest of the session instead of just once, which made Claude seem forgetful and repetitive."* — *April 23 postmortem* (2026-04-23). Pin `effort: high` explicitly in agents that need consistent reasoning depth — don't rely on session defaults.

---

## 6. Agent Patterns from Shipped Plugins (2026)

These lessons come from two production Claude Code plugins (`dbt-pipeline-toolkit` and `fabric-dataflow-migration-toolkit`) that shipped through external marketplaces and were tested on fresh installs. Each entry is a constraint that **was discovered the hard way** — works in dev, silently breaks at install. Cross-references point back to the originating finding (DBT F#, Fabric N#) in the plugins' own `plugin_learnings.md`.

The companion `claude-code-plugin-best-practices.md` covers the plugin-packaging side. This section is strictly about how those constraints shape `agent.md` authoring.

### 6.1 Agent namespacing diverges by directory layout

The registered name of a plugin agent depends entirely on whether you use a **flat** or **subdirectory** layout under `agents/`. The official docs example shows the flat case; everything else is undocumented.

| Layout | File | Registered name | Segments |
|---|---|---|---|
| Flat | `agents/<name>.md` | `<plugin>:<name>` | 2 |
| Subdirectory | `agents/<name>/agent.md` | `<plugin>:<subdir>:<frontmatter-name>` | 3 |

If you use subdirectories (most plugins do), the registered name has **three** segments, and the subdirectory name should match the frontmatter `name` field exactly. If they diverge, the registered name silently follows the directory.

Every cross-reference must use the full registered name on a fresh install:

```yaml
# Orchestrator frontmatter
tools: Agent(my-plugin:business-analyst:business-analyst, my-plugin:data-explorer:data-explorer), Read, Bash
```

```python
Task(subagent_type: "my-plugin:dbt-staging-builder:dbt-staging-builder", prompt: "...")
```

```bash
claude --agent my-plugin:orchestrator:orchestrator "Build a pipeline"
```

The `/agents` picker on a freshly-installed plugin is the source of truth. Verify there, not in dev mode. *(DBT F1 — no public release note confirms the 3-part format; consistent with `/plugin` discovery surface since v2.1.145.)*

### 6.2 Plugin-shipped agents lose `permissionMode`, `hooks`, `mcpServers` in frontmatter

Three frontmatter fields are silently stripped from plugin-shipped agents at install time as a security boundary — they remain valid on standalone `~/.claude/agents/*.md` files but no longer load when shipped via a plugin.

| Field | Plugin agent frontmatter | Plugin-level (`plugin.json`) |
|---|---|---|
| `hooks` | Stripped | Supported |
| `mcpServers` | Stripped | Supported |
| `permissionMode` | Stripped | (not a `plugin.json` field) |

**Push permission to the call site:**

```python
Task(
  subagent_type: "my-plugin:builder:builder",
  prompt: "...",
  run_in_background: true,
  mode: "acceptEdits"      # passed by parent — NOT stripped
)
```

**Push hooks and MCP servers up to `plugin.json`.** *(DBT F2 — corroborated by v2.1.147 changelog "Fixed plugin agents declaring multiple Agent(...) types in tools: frontmatter dropping all but last entry" and the plugins-reference explicit security note.)*

### 6.3 Background subagents cannot answer permission prompts

A background subagent has no interactive channel — any tool call that triggers a permission prompt stalls indefinitely. Always set `mode` when spawning with `run_in_background: true`:

```python
Task(
  subagent_type: "my-plugin:profiler:profiler",
  prompt: "...",
  run_in_background: true,
  mode: "acceptEdits"
)
```

Know what `acceptEdits` actually covers. It auto-approves:
- `Write`, `Edit`, `MultiEdit` in the working directory
- **Filesystem Bash only**: `mkdir`, `touch`, `mv`, `cp`, `rm`, etc.

It does **not** auto-approve `python ...`, `dbt run`, `git ...`, `fab ...`. For those, you need a plugin-level `PreToolUse` hook with a narrow allowlist. *(DBT F3, F9.)*

### 6.4 Orchestrators only delegate from the main thread

Subagents cannot spawn subagents. If your orchestrator is auto-invoked from an existing session, becomes part of a `Task(...)` call, or is hosted by a slash command, it runs **as a subagent** and its `Agent(...)` tool is inert. Every `Task(...)` call it makes silently no-ops.

Document the canonical invocation prominently in agent body AND README:

```bash
cd <target-repo>
claude --agent my-plugin:orchestrator:orchestrator "Start the pipeline"
```

Do not ship a slash command that hosts an orchestrator-that-delegates. *(DBT F4, Fabric N11 — corroborated by v2.1.116 "Agent `hooks:` fire as main-thread.")*

### 6.5 `AskUserQuestion` is NOT available in subagents — at all

This is a hard restriction. Per the SDK docs: *"AskUserQuestion is not currently available in subagents spawned via the Agent tool."* Foreground vs. background is irrelevant. Listing `AskUserQuestion` in a subagent's `tools:` has no effect.

**Use the three-step JSON-envelope pattern** instead of "interactive subagent":

```
Subagent (Mode: analyze, background)
   └─> writes refactor-questions.json with applicable questions + options

Parent (orchestrator main thread)
   └─> reads envelope, calls AskUserQuestion, writes refactor-answers.json

Subagent (Mode: write, background)
   └─> reads answers, produces side-effecting outputs
```

This removes a class of silent-failure bugs where subagents fall back to defaults when `AskUserQuestion` doesn't exist. *(Fabric N15. Corroborated by Issues #18721, #20275, #34592. v2.1.147 fixed `AskUserQuestion` suppression in auto mode in the **main thread** — the subagent restriction has never been acknowledged in a release note.)*

### 6.6 Interactive subagents that legitimately need foreground

For interactive tools *other than* `AskUserQuestion` (e.g. permission prompts that should pass through to the user), spawn with explicit `run_in_background: false`:

```python
# WRONG — comments are not parameters
Task(
  subagent_type: "my-plugin:migration-analyst:migration-analyst",
  prompt: "...",
  # foreground — needs interactive access
)

# RIGHT
Task(
  subagent_type: "my-plugin:migration-analyst:migration-analyst",
  prompt: "...",
  run_in_background: false,
  mode: "acceptEdits"
)
```

After the foreground subagent returns, the orchestrator must **verify the user-input artifact** actually contains real user input — not defaults. If empty or default-only, halt rather than letting the pipeline drift past unattended. *(Fabric N9.)*

### 6.7 Plan-mode approval requires `EnterPlanMode` — or use `AskUserQuestion` instead

A stage instruction that says "enter native plan mode for user approval" silently no-ops if `EnterPlanMode`/`ExitPlanMode` are not in the orchestrator's `tools:`. No error — agent continues, skipping the approval gate.

| Approach | Trade-offs |
|---|---|
| Add `EnterPlanMode, ExitPlanMode` to orchestrator `tools:` | Native UX. Feature availability varies by Claude Code version. Silent no-op on miss. |
| Use `AskUserQuestion` with `Approve` / `Revise` / `Abort` options | Portable across versions, depends only on a tool the orchestrator already has |

The `AskUserQuestion` variant is safer for plugins:

```
1. Print the plan/summary as a text block (full context).
2. AskUserQuestion: ["Approve and proceed", "Revise <stage-name>", "Abort"]
3. Branch the orchestrator based on the chosen label.
```

Note: when plan mode is active (via Shift+Tab) and the orchestrator uses `Agent`, the spawned subagents are **not subject to the plan-mode constraint** — they can freely make edits. (Issue #43777, open as of v2.1.150.) *(Fabric N10.)*

### 6.8 Worktree isolation can destroy newly-written files

`isolation: worktree` is appealing for parallel builders, but interacts badly with agents that **create new files** (notebooks, generated docs, build artifacts). The default WorktreeRemove hook runs `git worktree remove --force`, which deletes every untracked file the agent just wrote. Envelopes still report `status: success`; the files are simply gone.

**Only adopt `isolation: worktree` when:**

1. The files the agent writes are already git-tracked (e.g. dbt model `.sql` files edited in place), AND
2. The WorktreeRemove hook explicitly commits and merges back to the main worktree **before** the `git worktree remove --force`.

The dbt plugin's worktree hooks work because dbt models are source-controlled text. The Fabric plugin's builders write new `.ipynb` files that were not yet tracked — worktree isolation destroyed every notebook. Resolution: drop the isolation and rely on unique per-agent filenames (`nb_{layer}_{query}.ipynb`).

If your agent creates new files of any kind, either fix the WorktreeRemove hook to merge back or don't use worktree isolation. *(Fabric N8.)*

### 6.9 Trust-but-verify every envelope from background builders

A subagent reporting `status: 'success'` with a populated `notebook_path` is **not enough evidence** that the file exists. Worktree removal, permission stalls, path mismatches, and silent-fallback bugs all produce "successful" envelopes with no real output.

```python
# In orchestrator LLM text after fan-out completes
for envelope in builder_envelopes:
    if envelope["status"] == "success":
        # Verify the artifact on disk — atomic Bash ls call
        ls envelope["notebook_path"]
        # If the ls fails, halt and surface as a deviation
```

Missing file → halt the pipeline and surface the gap as a deviation. *(Fabric N8.)*

> "Write extremely high-quality tests… Claude will work autonomously to solve whatever problem I give it. So it's important that the task verifier is nearly perfect, otherwise Claude will solve the wrong problem." — *Building a C compiler with parallel Claudes* (2026-02-05)

### 6.10 Background subagents cannot read plugin-cache paths

A background subagent runs with restricted filesystem permissions and cannot read files outside the project's working directory. That includes the plugin cache — anything under `~/.claude/plugins/cache/.../reference/...` is unreachable.

| Pattern | When |
|---|---|
| **Copy at scaffold time** — project initializer copies plugin reference materials into the project once at scaffold time. Subagent prompts use project-local paths from then on. | References are small and stable. (default) |
| **Inline in prompt** — orchestrator reads the plugin-cache file itself (its permissions are unrestricted as the main thread) and embeds the content in the subagent's prompt string. | References are very small. |
| **Foreground escape hatch** — spawn the consumer in foreground (it inherits the orchestrator's read scope). | Other patterns don't apply. Defeats parallel fan-out. |

Validate the scaffolded folder by a **sentinel file**, not just `dir.exists()`. A half-populated folder passes naive existence checks but fails at runtime. *(Fabric N14.)*

Related: Issue #38699 — `CLAUDE_PLUGIN_ROOT` is inconsistent between hooks/skills and agent environment for local marketplace plugins. Open as of v2.1.150.

### 6.11 Stage ordering must match write/read dependency order

If Stage N reads a file Stage M creates, M must run before N. Filesystem commands like `cp`, `mkdir`, `mv` auto-create parent directories, so a stage that *writes into* a folder that doesn't exist doesn't fail loudly — it creates ad-hoc duplicate folder structures and downstream stages can't find anything.

**Always scaffold first.** Lint stages so no read path appears before its corresponding write path. *(Fabric N14 secondary.)*

### 6.12 Atomic Bash commands only

Every Bash tool call — whether the orchestrator runs it directly or generates it inside plugin/skill/agent/hook code — must be a single atomic operation. This is the single highest-impact reliability rule for background-subagent orchestration.

**Forbidden in every Bash command:**
- `&&` or `||` (logical chain)
- `;` (sequential chain)
- `|` or `|&` (pipe)
- Backgrounding with `&`
- Subshells `(...)`
- Command substitution `` `...` `` or `$(...)`
- Heredocs (`<<EOF`)
- Non-essential redirects `2>/dev/null` / `>/dev/null` — Claude Code's Bash tool handles exit codes and stderr natively
- `cd <path> && <command>` — use the Bash tool's implicit CWD or pass `-C <path>` to the binary

**Allowed exception:** literal operators inside a quoted string the shell will not interpret — `python foo.py --sql "SELECT a || b FROM t"`.

**Where conditional/sequential logic belongs:** in the LLM's text between calls, not in shell pipelines. Issue multiple Bash tool calls, read each output, choose the next command. If a step genuinely cannot be expressed as a single atomic command, write a Python script and call it as one atomic invocation.

**Why:** Claude Code's permission layer matches rules per subcommand. Compound expressions fall through to interactive prompts. Background subagents have no channel for those prompts and stall silently. Atomic filesystem commands also auto-approve under `acceptEdits` natively.

**Cost:** roughly 1–3% more tokens per session from extra tool-call protocol overhead. Cheap for the reliability gain. *(DBT F9 Round 2 — community blind spot, not discussed in any surveyed video.)*

### 6.13 Inline the atomic-Bash rule in every agent body that runs Bash

Plugin-shipped subagent CLAUDE.md loading is unreliable (project CLAUDE.md may not exist yet during early stages; plugin-subagent context assembly may strip body content per Issues #13605/#13627). The agent's own body is the only context guaranteed to load at spawn time.

| Audience | Reads | Why the rule must be there |
|---|---|---|
| Orchestrator during build | Its own `agent.md` body | Governs stages before any project CLAUDE.md exists |
| Specialist subagents during build | Their own `agent.md` body | Governs them regardless of plugin-subagent CLAUDE.md loading |
| Post-build sessions in the project | Project `CLAUDE.md` template | Governs ongoing maintenance |
| Every session on the machine | Global `~/.claude/CLAUDE.md` | Cross-project enforcement |

Yes, the rule ends up duplicated (~60 lines / ~1.2K tokens across all files). The cost of missing it in any one audience is silent stalls. *(DBT F9 Round 3.)*

### 6.14 `skills:` frontmatter on agents uses the 2-part namespace

Agents and skills do not share a namespace shape. Plugin **agents** in subdirectory layout are 3-part (`<plugin>:<subdir>:<name>`); plugin **skills** are always 2-part (`<plugin>:<skill-name>`) because skills are flat single-directory units.

```yaml
# WRONG — bare names resolve to nothing in plugin context
skills: dbt-runner, data-profiler

# WRONG — 3-part form is for agents, not skills
skills: my-plugin:dbt-runner:dbt-runner

# RIGHT — 2-part form for skill preloading
skills: my-plugin:dbt-runner, my-plugin:data-profiler
```

Bare names produce an empty preload silently. *(DBT F8 — subagents previously failed to inherit skill context entirely; fixed in v2.1.133 "Fixed subagents not discovering skills.")*

### 6.15 Builders that produce output files need explicit "don't pollute" instructions

Background builders consistently produce extra files outside their declared output path unless explicitly forbidden. From a live run of 14 parallel bronze builders:

| Pollution type | Count | Cause |
|---|---|---|
| `.py` duplicate next to `.ipynb` | 1 | Agent body still prescribed `.py` naming convention |
| Builder reports in `1 - Documentation/*.md` | 3 | Agent body said "Save documentation to `1 - Documentation/`" |
| Loose JSON envelopes on disk | 3 | Orchestrator prompt said "Return JSON envelope" — ambiguous |
| `NOTEBOOK_BUILD_REPORT_*.md` files | 1 | Agent body's "Phase 6: Report" generalized into a file |
| Invented top-level directories | 2 | Pure hallucination — not in scaffold or prompts |

**Builder agent bodies must include explicit forbidden behaviors:**
- No files outside the declared `notebook_path` / `model_path` / output path.
- No writes to orchestrator-owned folders.
- No invented top-level directories. If a folder isn't in the scaffold, don't create it.
- No build-report files. The Completion Summary template is for your chat response, not a file.
- No envelope-to-disk. Specify exactly where the envelope goes.

**Orchestrator's spawn prompt must be unambiguous about "return":**

```
WRONG: "Return JSON envelope: { status, notebook_path, ... }"
RIGHT: "Include this JSON envelope as the LAST block of your chat response —
       formatted as a fenced ```json``` block. DO NOT write the envelope to a file.
       DO NOT create any files outside the single notebook_path specified above."
```

A pre-shipment audit gate that greps agent bodies for forbidden naming patterns / forbidden write paths catches drift before it ships. *(Fabric N17.)*

### 6.16 Don't invent CLI flags mid-refactor

When collapsing compound shell expressions into single atomic invocations, the temptation is to write `python script.py --fail-below 80` and document it — even if `--fail-below` doesn't exist. An LLM (or a tired human) will happily invent a plausible flag and ship docs for a feature that doesn't exist.

**Mitigations:**
- Read the script before documenting any flag you reference in `agent.md`, `SKILL.md`, or orchestrator prompts.
- Add a pre-commit / pre-shipment audit that greps `agent.md` and `SKILL.md` for flags in shell commands and validates them against the script source.

Corollary: when a subagent invokes a script with an enforcement-mode default (e.g. `--target 80` exits 1 below threshold), pass the explicit value so the orchestrator can distinguish a real validation failure from a spurious exit code. *(DBT F9 Round 2.5.)*

---

## 7. Multi-Agent Orchestration Patterns

### 7.1 Pattern catalog

| # | Pattern | Use |
|---|---|---|
| A | **Sequential Pipeline** | research → plan → implement → test → review; each stage hands summary to next |
| B | **Parallel Fan-Out** | Independent subtasks spawned simultaneously; only when no file overlap |
| C | **Map-Reduce** | N workers + 1 aggregator (e.g. `/simplify`: 3 reviewers in parallel → aggregator applies fixes) |
| D | **Dependency-Aware Sequential** | Shared task list with explicit deps; blocked tasks auto-unblock (Agent Teams only) |
| E | **Handoff / Ralph Loop** | Stateless-but-iterative: pick task → implement → validate → commit → reset context. Used in Anthropic's C-compiler experiment (16 parallel agents). Prevents context rot. |
| F | **Hierarchical Delegation** | 2 feature leads each spawn 2-3 specialists. Keeps parent context clean. Requires Agent Teams. |
| G | **Competing Hypotheses (Debate)** | N teammates each investigate a different theory, actively disprove others; surviving theory wins. |

Sam Witteveen's videos and Claude Architect's orchestration deep-dive surface a similar 6-pattern taxonomy: orchestrator, fan-out, validation chain, specialist routing, hierarchy, watchdog.

> "Multi-agent systems work mainly because they help spend enough tokens to solve the problem." — *How we built our multi-agent research system* (2025-06-13)

### 7.2 Orchestrator prompt skeleton

```markdown
---
name: orchestrator
description: Delegates multi-stage implementation tasks. MUST BE USED for any task touching 3+ files or spanning research→plan→implement→test.
tools: Agent(researcher, planner, implementer, tester, reviewer), Read, TodoWrite
model: sonnet
---

You are an orchestrator. You DO NOT write code directly.

For every task:
1. Spawn `researcher` to gather context (read-only Explore equivalent).
2. Read its summary. If ambiguous, spawn `planner` to produce a plan.
3. Surface the plan via ExitPlanMode (or AskUserQuestion) and wait for user approval.
4. Spawn `implementer` with isolation: worktree for risky changes.
5. Spawn `tester` to verify; if fails, re-spawn `implementer` with feedback.
6. Spawn `reviewer` last to validate the final diff.

Never skip stages. Never write code yourself. Summaries only.
```

### 7.3 Three-step parent-owned interactive pattern

Because `AskUserQuestion` doesn't work in subagents (§6.5), and because parallel work can't pause for user input, codify the parent-owned interaction pattern as your *first* design, not the fallback:

```
1. Parent spawns subagent in Mode: analyze (background). Subagent writes
   questions.json with applicable questions + options.
2. Parent reads envelope, calls AskUserQuestion in the main thread,
   writes answers.json.
3. Parent spawns subagent in Mode: write (background). Subagent reads
   answers, produces side-effecting outputs.
```

### 7.4 Trust-but-verify after fan-out

Always verify artifacts on disk after each fan-out stage — `status: success` envelopes alone are not enough (§6.9).

### 7.5 User interaction budget

Treat user questions as a scarce resource. From the *Code Beats Markdown* and *Effective harnesses for long-running agents* posts: every "what should I do here?" interrupt drains a budget. Aggregate questions across stages into a single `AskUserQuestion` call where possible. Use dynamic question selection — only ask what the subagent's analysis actually surfaced as ambiguous.

### 7.6 Shared-state handoff options

| Mechanism | Use |
|---|---|
| **TodoWrite / task list** | Ordered task state visible to all teammates (Agent Teams) |
| **Plan mode + ExitPlanMode** | Read-only exploration locks edits until user approves the plan |
| **Filesystem handoffs** | Write plan to `_Plan/xyz.md`; next agent reads it |
| **Agent memory (`memory:` frontmatter)** | Persists across sessions (`user`/`project`/`local` scope) — first 200 lines / 25KB of MEMORY.md auto-injected |
| **CLAUDE.md hierarchy** | Global → project → folder; teammates load from CWD |

### 7.7 Worktree isolation in orchestration

Use for risky/exploratory edits, parallel agents that would otherwise conflict, competing implementations (A/B compare). Clean worktrees auto-delete; modified ones persist for review. **Subject to §6.8 caveats.**

`v2.1.143` added `worktree.bgIsolation: 'none'` setting for background sessions to edit working copy directly without `EnterWorktree`.

### 7.8 Agent Teams (experimental)

Enable: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Requires Claude Code v2.1.32+.

**SendMessage tool message types:**

| Type | Use |
|---|---|
| `message` | Direct message to one teammate |
| `broadcast` | To all teammates — cost scales with team size |
| `shutdown_request` | Ask teammate to gracefully exit |
| `plan_approval_response` | Lead approves/rejects a teammate's plan |

Messages land in `~/.claude/teams/{team-name}/inboxes/{name}.json`. Stopped agents auto-resume on message receipt.

**Display modes:** `in-process` (default), `tmux` / split panes.

### 7.9 Team sizing

- **3-5 teammates** is the sweet spot (Anthropic docs + Boris Cherny's workflow: 5 parallel sessions across 5 terminals).
- **5-6 tasks per teammate** max.
- 16+ agents requires external oracle + file-level locking + Ralph loop (compiler experiment).

> "With agent teams, multiple Claude instances work in parallel on a shared codebase without active human intervention." — *Building a C compiler with parallel Claudes*

### 7.10 Harness design lessons

> "Harnesses encode assumptions that go stale as models improve." — *Scaling Managed Agents: Decoupling the brain from the hands* (2026-04-08)

> "Context resets — clearing the context window entirely and starting a fresh agent, combined with a structured handoff — addresses both these issues." — *Harness design for long-running application development* (2026-03-24)

> "Separating the agent doing the work from the agent judging it proves to be a strong lever to address self-evaluation issues." — same post

The planner / generator / evaluator split (GAN-inspired) is the highest-leverage harness pattern for multi-hour tasks.

---

## 8. Common Agent Anti-Patterns

Single consolidated list. Cross-references point to the section with the full explanation.

**Orchestration shape:**
- **Context pollution** — orchestrator does exploration itself, then delegates; duplicate work
- **Over-delegation** — 8 micro-agents when 2 would do; coordination overhead > benefit
- **Line-level task splits** — two agents editing same file = overwrites; split by file or feature
- **Monolithic tasks** — all N agents hit the same bug and overwrite each other's fixes
- **Lead does the work** — lead implements instead of waiting; tell it "wait for teammates"
- **No Definition of Done** — agents stop prematurely or over-build
- **Missing test oracle** — without reliable tests, agents solve the wrong problem (*"Claude will solve the wrong problem"* — C-compiler post)
- **Prompt glue chaining** — embedding handoff instructions in prompts; use hooks/files instead

**Configuration:**
- **Tool sprawl** — omitting `tools:` grants every tool
- **Massive prompts trying to handle everything** — specialization wins
- **Files over 2K lines** — rules at the bottom get lost
- **Test-only focus at expense of general solutions** — agents game tests
- **Skipping human review gates** — production changes need approval

**Background/permissions (plugin-specific):**
- **Interactive agents in background mode** — stalls silently (§6.3, §6.5)
- **Compound Bash with `&&`/`;`/`|`** — background subagents stall on permission prompts (§6.12)
- **`isolation: worktree` for builders that write new files** — default WorktreeRemove deletes them (§6.8)
- **Trusting `status: success` envelopes without disk verification** (§6.9)

**Namespacing:**
- **Bare names in `skills:` frontmatter** — empty preload silently (§6.14)
- **3-part name in `skills:` frontmatter** — wrong namespace shape (§6.14)
- **Plugin agents with `hooks:`/`mcpServers:`/`permissionMode:`** — silently stripped (§6.2)
- **Subagent prompts referencing `${CLAUDE_PLUGIN_ROOT}/reference/`** — unreachable from background subagent (§6.10)
- **Multiple separate `Agent(...)` entries in `tools:` (pre-v2.1.147)** — all but last dropped silently

---

## 9. External Source Validation

Most quotes are embedded inline above. This table is the catch-all for citations that didn't fit naturally elsewhere — primarily Anthropic CHANGELOG entries and GitHub issue trackers for documented-but-unannounced behaviors.

> **A note on "DBT F9 Round N" sub-citations.** The source `dbt-pipeline-toolkit/_Documentation/plugin_learnings.md` has a single heading "Finding 9" — there are no "Round 2 / 2.5 / 3" sub-headings in the source file. The round suffixes here and in `claude-code-plugin-best-practices.md` denote distinct *follow-up incidents* uncovered while iterating on the original F9 fix: Round 2 = compound-Bash regression after the initial allowlist hook landed; Round 2.5 = SKILL.md documenting a CLI flag that didn't exist on the script; Round 3 = the "duplicate the atomic-Bash rule across all audiences" decision. These are our own naming convention for traceability — when reading the source, all three trace back to Finding 9's narrative.

| Topic | Source |
|---|---|
| Plugin agent frontmatter stripping | [plugins-reference](https://code.claude.com/docs/en/plugins-reference) — "For security reasons, `hooks`, `mcpServers`, and `permissionMode` are not supported for plugin-shipped agents." |
| Multi-`Agent(...)` drop bug fix | CHANGELOG v2.1.147 |
| Subagent skill discovery fix | CHANGELOG v2.1.133 — "Fixed subagents not discovering skills." |
| `--agent` honors `permissionMode` | CHANGELOG v2.1.119 |
| Agent hooks main-thread only | CHANGELOG v2.1.116 — "Agent `hooks:` fire as main-thread." |
| `subagent_type` matching tolerant | CHANGELOG v2.1.140 |
| `worktree.bgIsolation: 'none'` | CHANGELOG v2.1.143 |
| `--agent` plugin discovery fix | CHANGELOG v2.1.143 |
| Effort default postmortem | [April 23 postmortem](https://www.anthropic.com/engineering/april-23-postmortem) (2026-04-23) |
| AskUserQuestion subagent restriction | Issues [#18721](https://github.com/anthropics/claude-code/issues/18721), [#34592](https://github.com/anthropics/claude-code/issues/34592) |
| Plan-mode subagent leakage | Issue [#43777](https://github.com/anthropics/claude-code/issues/43777) (open as of v2.1.150) |
| Task-tool custom-agent Not Planned | Issue [#19276](https://github.com/anthropics/claude-code/issues/19276) (works anyway post-v2.1.143) |
| `CLAUDE_PLUGIN_ROOT` inconsistency | Issues [#27145](https://github.com/anthropics/claude-code/issues/27145), [#38699](https://github.com/anthropics/claude-code/issues/38699) (open) |
| Worktrees-per-agent origin | IndyDevDan, 2025-05-26 — [video](https://www.youtube.com/watch?v=f8RnRuaxee8) |
| Six-pattern sub-agent taxonomy | "How To Use Sub-Agents Better Than 99% of People" — [video](https://www.youtube.com/watch?v=6mG6tS6WG00) |
| Code beats markdown for transformations | Sam Witteveen — [video](https://www.youtube.com/watch?v=IjiaCOt7bP8) |
| Boris Cherny's vanilla setup | Pragmatic Engineer interview — 5 parallel sessions, plan-mode-first, team CLAUDE.md committed |
| Hooks as deterministic guarantee | *Best practices for Claude Code* — "Hooks are deterministic and guarantee the action happens." |

---

## 10. Further Reading

### Companion docs
- `how-to-create-agents.md` — step-by-step build guide
- `how-to-create-skills.md` — step-by-step skill guide
- `how-to-create-hooks.md` — step-by-step hook guide
- `how-to-create-plugins.md` — step-by-step plugin guide
- `claude-code-skill-best-practices.md` — skills patterns
- `claude-code-hooks-best-practices.md` — hooks patterns
- `claude-code-plugin-best-practices.md` — plugin packaging

### Top-5 agent-relevant blog posts (with URLs)

1. *Building agents with the Claude Agent SDK* — https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk
2. *Building effective agents* — https://www.anthropic.com/engineering/building-effective-agents
3. *How we built our multi-agent research system* — https://www.anthropic.com/engineering/multi-agent-research-system
4. *Building a C compiler with parallel Claudes* — https://www.anthropic.com/engineering/building-c-compiler
5. *Effective context engineering for AI agents* — https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

### Top-5 agent-relevant videos

1. **Claude Agent SDK [Full Workshop] — Thariq Shihipar (Anthropic)** — https://www.youtube.com/watch?v=TqC1qOfiVcQ
2. **How To Use Claude Code Sub-Agents Better Than 99% of People** (six-pattern taxonomy) — https://www.youtube.com/watch?v=6mG6tS6WG00
3. **Claude 4 ADVANCED AI Coding: Parallelize Claude Code with Git Worktrees — IndyDevDan** — https://www.youtube.com/watch?v=f8RnRuaxee8
4. **#412 Max: The Boris Cherny Workflow** (vanilla, plan-mode-first, 5 parallel sessions) — https://www.youtube.com/watch?v=QZCJ4aEhrMk
5. **Agent Skills: Code Beats Markdown — Sam Witteveen** (script for transformation, markdown for judgment) — https://www.youtube.com/watch?v=IjiaCOt7bP8

### Primary documentation
- [Claude Code Sub-agents](https://code.claude.com/docs/en/sub-agents)
- [Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [Claude Code Plugins Reference](https://code.claude.com/docs/en/plugins-reference)
- [Claude Code Best Practices (live)](https://code.claude.com/docs/en/best-practices)
- [Agent Skills best practices](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/best-practices)
- [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
