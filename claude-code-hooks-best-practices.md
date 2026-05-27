# Claude Code Hooks — Best Practices

**Last Updated:** 2026-05-25
**Sources:** Official Anthropic hooks reference, best-practices live doc, plugins reference; Claude Code release notes v2.1.111–v2.1.150; expert YouTube creators (IndyDevDan, Greg Baugues, John Lindquist, Sam Witteveen); production-plugin lessons from `dbt-pipeline-toolkit` and `fabric-dataflow-migration-toolkit`.

> **Companion documents in this folder:**
> - `how-to-create-hooks.md` — step-by-step mechanics (event catalogue, JSON contract, worked examples)
> - `claude-code-agent-best-practices.md` — agent design patterns
> - `claude-code-skill-best-practices.md` — skill design patterns
> - `claude-code-plugin-best-practices.md` — plugin packaging patterns
> - `how-to-create-plugins.md` — plugin step-by-step guide

This document is **patterns and rationale**. For the underlying mechanics (event catalogue, JSON I/O, exit codes, scopes), read `how-to-create-hooks.md` first.

---

## Table of Contents

1. [Configuration Best Practices](#1-configuration-best-practices)
2. [Plugin-Level Hooks — Patterns from Shipped Plugins (2026)](#2-plugin-level-hooks--patterns-from-shipped-plugins-2026)
3. [Logging and Observability Best Practices](#3-logging-and-observability-best-practices)
4. [Security & Permission Best Practices](#4-security--permission-best-practices)
5. [Performance Best Practices](#5-performance-best-practices)
6. [Hook Event Selection Guide](#6-hook-event-selection-guide)
7. [Stop Hook Best Practices](#7-stop-hook-best-practices)
8. [Subagent Monitoring Best Practices](#8-subagent-monitoring-best-practices)
9. [Session Management Best Practices](#9-session-management-best-practices)
10. [Windows-Specific Best Practices](#10-windows-specific-best-practices)
11. [Testing and Debugging Best Practices](#11-testing-and-debugging-best-practices)
12. [Anti-Patterns to Avoid](#12-anti-patterns-to-avoid)
13. [External Source Validation (Citations)](#13-external-source-validation-citations)
14. [Further Reading](#14-further-reading)

---

## 1. Configuration Best Practices

### Use the right scope

| Scope | File | When to use |
|-------|------|-------------|
| **User-global** | `~/.claude/settings.json` | Personal habits (notifications, logging, safety) |
| **Project-shared** | `.claude/settings.json` | Team standards (formatting, protected files, CI checks) |
| **Project-local** | `.claude/settings.local.json` | Personal project overrides (gitignored) |
| **Plugin** | `plugin.json` `hooks` block | Hooks bundled with a plugin |

**Rule of thumb:** if it enforces a team standard, put it in project settings. If it's personal preference, put it in user settings. If it ships as part of a reusable bundle, put it in `plugin.json`.

### Keep hook scripts external

Don't inline complex logic into the JSON `command` field — it becomes unreadable and unmaintainable. Reference a script file instead:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "python \"$CLAUDE_PROJECT_DIR/.claude/hooks/validate-bash.py\""
      }]
    }]
  }
}
```

Place scripts in `.claude/hooks/` within the project for shareability, or `~/.claude/hooks/` for personal global hooks.

### Use `statusMessage` for clarity

Always provide a `statusMessage` so the status-line spinner tells the user what's happening:

```json
{
  "type": "command",
  "command": "python .claude/hooks/log-action.py",
  "statusMessage": "Logging agent action..."
}
```

Without `statusMessage`, the user sees only a generic spinner — when a hook stalls, they cannot tell which hook is responsible. With it, debugging a stuck or slow hook collapses from "what's running?" to "OK, the Bash allowlist hook is hung — let me look at that script."

### Prefer exec-form args over shell-string concatenation (v2.1.139+)

```json
{
  "type": "command",
  "command": "python",
  "args": ["${CLAUDE_PLUGIN_ROOT}/hooks/validate.py", "--strict"]
}
```

Eliminates shell-escape headaches when arguments contain spaces or quotes.

---

## 2. Plugin-Level Hooks — Patterns from Shipped Plugins (2026)

This section captures hook-design lessons learned from shipping two production Claude Code plugins (`dbt-pipeline-toolkit` and `fabric-dataflow-migration-toolkit`). The two plugins together ship 5 hook event registrations (PreToolUse Bash allowlist, PreToolUse Write/Edit structural validator, SessionStart config-check, plus 2 worktree hooks that were later removed). Every finding below comes from something that actually broke on a fresh install — not theory.

### 2.1 Plugin-level hooks vs. agent-level hooks

Plugin-shipped agents **cannot** declare `hooks:`, `mcpServers:`, or `permissionMode:` in their own `agent.md` YAML frontmatter — those fields are silently stripped at install time. But the plugin manifest (`plugin.json`) **can** ship hooks at the plugin level, and they work normally. The asymmetry is a deliberate security boundary: the plugin manifest is the user-auditable surface (visible in marketplace, greppable in repo, surfaced in `/plugin` inspection), while agent frontmatter is an unaudited spawn-time surface that would allow privilege escalation if it could declare hooks. (Source: DBT F2.)

> *"For security reasons, `hooks`, `mcpServers`, and `permissionMode` are not supported for plugin-shipped agents."* — `code.claude.com/docs/en/plugins-reference`

**Rule:** push every hook to `plugin.json`. Never rely on `hooks:` in an agent's frontmatter — it will not load.

```json
{
  "hooks": {
    "PreToolUse": [
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
    ]
  }
}
```

### 2.2 PreToolUse allowlist hook — the canonical pattern for unblocking background subagents

`acceptEdits` mode is more limited than its name implies: per the permissions reference, it only auto-approves **filesystem Bash** (`mkdir`, `touch`, `mv`, `cp`, `rm`) plus `Write`/`Edit`. Arbitrary Bash like `python profile_data.py`, `dbt run`, `fab job ...`, `git commit -m ...`, `az account ...` still flows through the normal permission layer. Background subagents (`run_in_background: true`) have no interactive channel, so the prompt goes nowhere and the subagent stalls with "I am unable to execute Bash commands." (Source: DBT F3 + F9.)

The fix is a PreToolUse hook at plugin level with `matcher: "Bash"` that emits a `permissionDecision: "allow"` response for narrowly-defined known patterns. The hook contract:

```python
# Hook reads PreToolUse JSON from stdin, emits one of:
#
#   {"hookSpecificOutput": {
#       "hookEventName": "PreToolUse",
#       "permissionDecision": "allow",
#       "permissionDecisionReason": "Plugin-internal python script"
#   }}
#
# Or empty {} to fall through to the default permission flow.
# Exit 0 always. Never exit 2 (that blocks the call unconditionally).
```

Concrete reference: `dbt-pipeline-toolkit/hooks/approve-plugin-bash.py` — the design comment block at the top of the file is excellent template material for any new plugin's allowlist hook.

### 2.3 Allowlist design rules

When you ship a PreToolUse allowlist hook, follow these rules to keep the security surface auditable (Source: DBT F9):

- **Narrow patterns**: require the full `${CLAUDE_PLUGIN_ROOT}/skills/<name>/scripts/<file>.py` shape. Never use bare `python *` — that broadly unlocks Python.
- **One auditable file**: all allowlist regex inline in one Python script (`hooks/approve-plugin-bash.py`), greppable at install time. Don't load the allowlist from an external config file or remote source.
- **Allow-only**: the hook can ALLOW but never DENY. Explicit `permissions.deny` rules set by the user still take precedence per the Claude Code permission layering. This is a feature, not a limitation — it keeps user control intact.
- **Default to fall-through**: unknown patterns return `{}` (empty object), which sends the call to the default permission flow. Do not return `"deny"` for unknown patterns — that surprises users.
- **Exit 0 always**: never exit 2 (that blocks the call unconditionally, regardless of the user's permission settings).
- **Atomic-commands assumption**: the normal case is a single atomic command per call. A compound-command splitter is defensive fallback code, not load-bearing. See next subsection.

```python
ALLOWLIST_PATTERNS = [
    # Plugin-internal Python scripts only
    re.compile(r'python\s+"?\$\{CLAUDE_PLUGIN_ROOT\}/skills/[\w-]+/scripts/[\w-]+\.py"?.*'),
    # dbt CLI subcommands the plugin actually uses
    re.compile(r'dbt\s+(run|test|build|parse|debug|seed|snapshot|compile|deps|list)\b.*'),
    # git commands used by scaffolding
    re.compile(r'git\s+(init|status|add|commit|log|show|diff|branch|rev-parse)\b.*'),
]
```

### 2.4 Atomic-commands rule — match the tool's grain

Every Bash command — generated by the orchestrator OR run interactively — must be a single atomic operation. No `&&`, `||`, `;`, `|`, subshells `(...)`, `$(...)`, backticks, or heredocs. The permission layer evaluates rules **per-subcommand**, so compound expressions force the layer to split-and-recheck — and any bug in the splitter is a security surface. Atomic commands also unlock the `acceptEdits` auto-approval path for filesystem operations natively, so you avoid touching the hook chain at all for `mkdir`/`touch`/`mv`/`cp`/`rm`. (Source: DBT F9 Round 2.)

The quote-aware compound-command splitter in `approve-plugin-bash.py` is defensive fallback for when a contributor accidentally ships a compound — it splits on operators while respecting single/double quotes — but the source code should never need it. Cost of the atomic-commands convention: roughly 1–3% more tokens per session from extra tool-call overhead. Benefit: the plugin works at all in background-orchestration mode.

**Rewrite patterns:**
- "run A then B" → two separate Bash tool calls in sequence
- "run A, if fails run B" → issue A, read exit code in LLM text, conditionally issue B as a second call
- "pipe A to B" → issue A, read output in LLM text, then issue B — OR put both in a Python script called atomically

### 2.5 Write/Edit structural validators

Use PreToolUse with `matcher: "Write|Edit"` to block file writes that violate plugin-specific structural rules. Example from `fabric-dataflow-migration-toolkit`: block `.py` writes under `3 - Notebooks/` because Fabric's notebook deploy API treats `.py` files as a single mega-cell — real notebooks must be `.ipynb` Jupyter JSON. (Source: Fabric N1 + N17.)

```python
# Inside validate-fabric-structure.py
if file_path.endswith(".py") and "3 - Notebooks/" in file_path:
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": (
                "Use .ipynb Jupyter JSON format — Fabric's notebook "
                "deploy API treats .py as a single mega-cell. See N1."
            )
        }
    }))
    sys.exit(0)
```

**Belt-and-suspenders pattern**: enforce the same rule in (a) the agent.md body, (b) the runtime hook, and (c) a pre-shipment audit gate. Each catches a different drift class — the agent might still prescribe the wrong extension (N17 happened because the bronze-builder agent.md still said `.py` even after N1 claimed it was fixed); the hook catches runtime violations; the audit gate catches authoring-time drift before the PR merges. All three layers together is overkill AND correct: each audience reads a different file.

### 2.6 SessionStart config-check hook

When `userConfig` may not have been prompted at install (DBT F5 Problem B — the install prompt sometimes silently no-ops on update or with certain plugin manifest shapes), ship a SessionStart hook that reads the plugin's options from `~/.claude/settings.json`, detects missing required values, and prints a setup-instructions block to stderr. The block surfaces in chat context via the session-start hook reminder mechanism.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/session-start-config-check.py"
          }
        ]
      }
    ]
  }
}
```

The Fabric plugin uses an empty `matcher: ""` so it runs on every session start (startup, resume, compact — all sources). The hook is idempotent: if all required config is set, it exits silently with code 0 and the user never sees anything.

### 2.7 Worktree hooks can silently destroy agent output

The dbt plugin originally shipped `WorktreeCreate`/`WorktreeRemove` hooks for parallel-fanout isolation: each builder agent runs inside its own `git worktree`, the hook commits-and-removes when the agent finishes. This works for dbt because models are tracked text — the builder commits its own files inside the worktree before exit, and `git worktree remove --force` is safe. (Source: DBT F8 — original design.)

But it **destroys output** for any builder that writes untracked artifacts: notebooks, generated docs, build outputs, profile JSONs, anything that hasn't been `git add`'d. The Fabric plugin's bronze-builder reported `status: 'success'` with a valid `notebook_path`, then the WorktreeRemove hook wiped the worktree filesystem milliseconds later — `.ipynb` files vanished before the orchestrator could see them. (Source: Fabric N8.)

**Decision rule for your plugin**: only adopt `isolation: worktree` if BOTH (a) the agent writes files that are already git-tracked AND (b) the WorktreeRemove hook explicitly commits and merges back to the main worktree before the `git worktree remove`. Default dbt-style hooks do NOT merge back — they assume the agent committed its work itself.

If your builders write untracked artifacts, either (a) add a commit-and-merge-back step to the WorktreeRemove hook, or (b) drop worktree isolation entirely and rely on unique per-agent filenames (`nb_bronze_{query}.ipynb` etc.) to prevent collisions. The Fabric plugin chose option (b) — removed `isolation: worktree` from both builder frontmatters, removed both worktree hooks from `plugin.json`, deleted the hook scripts.

**Grep your own plugin to detect the risk:**

```bash
grep -r "isolation: worktree" agents/
grep -r "WorktreeRemove\|WorktreeCreate" .claude-plugin/plugin.json
```

If either matches AND the agent body writes anything other than `.sql`, `.md`, or other typically-tracked text files, you're at risk.

### 2.8 Hook auditability is a design requirement

Plugins are user-installable code with privileged hook access. The plugin manifest plus every hook script must be greppable at install time so users can audit the allowlist before enabling the plugin. (Source: DBT F2.) Patterns:

- **Keep allowlist regex inline in the script** — not loaded from an external config or remote URL. Users can read the patterns directly in `hooks/approve-plugin-bash.py`.
- **Inline-document the security boundary** in the script's docstring. The `approve-plugin-bash.py` docstring states: "The hook can only ALLOW calls — it cannot override explicit `permissions.deny` rules the user has set." That's the contract; put it where reviewers read.
- **Cite explicit Claude Code docs URLs** in the docstring for any permission-related claim (e.g., the `acceptEdits` semantics, the per-subcommand evaluation rule). Reviewers can verify the claim against the docs in minutes.
- **No environment-dependent allowlists**: don't read `$ALLOWLIST_PATTERNS` from the environment or a settings file. The allowlist must be the same on every install.

### 2.9 `statusMessage` improves debuggability for permission hooks

When a hook is doing permission gating (allow/deny), set a clear `statusMessage` so the spinner surfaces it. "Checking plugin Bash allowlist..." is dramatically faster to debug than running silently — when a user reports "my command didn't run", a visible spinner immediately points at the right hook. (Source: DBT pattern, both plugins use this.)

```json
{
  "type": "command",
  "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/approve-plugin-bash.py",
  "statusMessage": "Checking plugin Bash allowlist..."
}
```

For Write/Edit structural validators, use a phrase that names the rule being checked: "Validating Fabric notebook structure..." or "Validating dbt file structure..." — the user sees instantly which rule fired.

### 2.10 Hook scripts must run on Windows AND Linux/macOS

Plugin hooks ship to wherever the user installs the plugin. Use Python's `pathlib`, avoid shell-specific syntax. (General principle reinforced by both plugins running on Windows/Git Bash and macOS/zsh.)

```json
"command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/approve-plugin-bash.py"
```

- Use `python`, NOT `python3` — Windows has `python.exe`, not `python3`. Python's launcher handles the version selection on both platforms.
- Use forward slashes inside `${CLAUDE_PLUGIN_ROOT}` paths — they work on Windows in Python and in most shell contexts. Backslashes break Git Bash double-quoted strings.
- Don't use bash-only constructs (`[[ ... ]]`, process substitution `<(...)`, `function() { }` syntax) inside hook scripts. Stick to Python where any logic is needed.
- Don't hardcode paths like `C:/Python313/python.exe` in plugin hooks — that works for global hooks (Section 10 below) but not for plugin hooks that ship to other machines.

### 2.11 Plugin audit gates as pre-commit checks

Build an audit script in `tests/` that validates the plugin before each release. Run it in CI; block PRs that fail. The audit catches drift between the rule (in agent.md body) and the runtime enforcement (hook). (Source: Fabric plugin's `tests/preshipment_audit.py`.)

The fabric plugin's 7 audit gates as concrete examples — each gate is a function returning pass/fail:

| Gate | What it validates |
|---|---|
| `gate_required_files` | All declared files (agents, skills, hooks, references) exist on disk |
| `gate_plugin_manifest` | `plugin.json` parses, has required fields, namespace format correct |
| `gate_paths` | All script invocations use `${CLAUDE_PLUGIN_ROOT}`, no `$HOME/.claude/skills/` |
| `gate_namespace` | Every `subagent_type:` reference uses the 3-part `<plugin>:<dir>:<name>` form |
| `gate_skills_frontmatter` | Agent `skills:` lines use 2-part `<plugin>:<skill>` form |
| `gate_atomic_bash` | No `&&`, `||`, `;`, `|`, `$(`, backticks in any Bash code block |
| `gate_notebook_extension` | No agent body prescribes `.py` notebook output under `3 - Notebooks/` |

The 7th gate (`gate_notebook_extension`) was added after Fabric N17 — the bronze-builder agent.md had drifted to prescribe `.py` even though the runtime hook blocked `.py`. The audit gate now catches this drift at PR time, so the agent.md body and the runtime hook can't fall out of sync again.

### 2.12 `InstructionsLoaded` event — useful for telemetry, not yet exploited

Not yet used by either shipped plugin, but worth knowing: the `InstructionsLoaded` event fires when a CLAUDE.md or `.claude/rules/*.md` file is loaded into context — both at session start and (per the docs) "when files are lazily loaded during a session." Useful applications:

- **Telemetry**: did my project's `CLAUDE.md` actually load into the orchestrator's context? Log each load event to confirm.
- **Drift detection**: if the loaded `CLAUDE.md` is missing a required rule, surface a warning via stderr at load time.
- **Dynamic context injection**: when a specific rules file loads, append related context (the dbt plugin could use this to inject the atomic-commands rule snippet when the project CLAUDE.md loads).

Worth piloting in a future plugin release — particularly for the DBT plugin's Stage 5 bootstrap (where `CLAUDE.md` doesn't exist in the target repo until the architecture-setup skill creates it, so the orchestrator's initial context assembly has no rules to load).

### 2.13 Known `CLAUDE_PLUGIN_ROOT` issues to work around

Open as of v2.1.150:

- Issue #27145: `CLAUDE_PLUGIN_ROOT` not set for SessionStart hooks.
- Issue #36585: `CLAUDE_PLUGIN_ROOT` not passed to UserPromptSubmit hooks.
- Issue #38699: `CLAUDE_PLUGIN_ROOT` inconsistent between hooks/skills and agent environment for local marketplace plugins — "for plugins installed from local marketplaces, `CLAUDE_PLUGIN_ROOT` is set to different values depending on context — hooks receive the source directory while the agent's shell environment receives the cache/install directory."

**Workaround**: derive the plugin root from `__file__` as a fallback:

```python
PLUGIN_ROOT = Path(os.environ.get("CLAUDE_PLUGIN_ROOT", Path(__file__).parent.parent))
```

---

## 3. Logging and Observability Best Practices

### Log structure — JSONL

Use **JSONL** (one JSON object per line) for machine-readable logs:

```json
{"timestamp":"2026-02-12T10:30:00Z","event":"PreToolUse","tool":"Bash","input":{"command":"npm test"},"session_id":"abc123"}
```

### What to log by event

| Event | What to capture |
|-------|----------------|
| `SessionStart` | Session ID, model, source (startup/resume/compact), timestamp |
| `UserPromptSubmit` | Prompt text (or hash if sensitive), timestamp |
| `PreToolUse` | Tool name, tool input summary, decision (allow/deny) |
| `PostToolUse` | Tool name, success status, duration, file paths affected |
| `PostToolUseFailure` | Tool name, error message, is_interrupt flag |
| `SubagentStart` | Agent ID, agent type, spawning prompt summary |
| `SubagentStop` | Agent ID, agent type, transcript path, `background_tasks`, `session_crons` |
| `Stop` | Whether `stop_hook_active`, session duration, `background_tasks`, `session_crons` |
| `SessionEnd` | Source (clear/logout/exit), total tool calls, session summary |

### File naming convention

```
_Agent Logs/
├── session-{date}-{session_id_short}.jsonl    # Per-session detailed log
├── sessions-index.jsonl                        # Session summary index
└── daily-{date}.jsonl                          # Daily aggregate (optional)
```

### Rotate and limit log size

- Cap individual session logs at 10 MB.
- Archive logs older than 30 days.
- Use daily aggregation to reduce file count.

### The IndyDevDan observability pattern

IndyDevDan's `claude-code-hooks-multi-agent-observability` repo demonstrates the canonical "hooks → WebSocket → dashboard" stack: 12 Python scripts intercept every lifecycle event, POST to a Bun HTTP server, persist to SQLite, stream to a Vue client via WebSocket. The point: *every plugin should make its hook output easily streamable to a dashboard like this*, even if you don't ship the dashboard.

> *"As we push into the age of agents, we need observability to scale our impact. We need to know what our agents are doing."* — IndyDevDan, "I'm HOOKED on Claude Code Hooks"

For your own plugins, ship JSONL log writes in PostToolUse / SubagentStop / SessionEnd, and document the schema. Users who want a dashboard can wire one up themselves.

---

## 4. Security & Permission Best Practices

### Always validate input

```bash
#!/bin/bash
INPUT=$(cat)
# Validate JSON is well-formed
echo "$INPUT" | jq empty 2>/dev/null || exit 0

# Extract fields safely
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty')
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
```

### Block dangerous operations

The **why:** PreToolUse hooks are the only deterministic gate between Claude and irreversible local damage (`rm -rf /`, `DROP TABLE`, raw disk writes). Treat the blocklist as a tripwire — exit 2 with a clear stderr message; the agent receives the rejection and can recover.

> **Mechanics:** see `how-to-create-hooks.md` § 6.1 for the canonical Bash recipe with the full pattern array.

### Protect sensitive files

The **why:** the Edit/Write tool surface is the most common path to accidentally exfiltrating `.env`, credentials files, or SSH keys into a code review or PR. A PreToolUse hook matching on `Edit|Write` catches this at the harness level, before the read even returns the file's content to Claude.

> **Mechanics:** see `how-to-create-hooks.md` § 6.2 for the canonical Bash recipe with the full protected-paths array.

### Quote everything, trust nothing

```bash
# GOOD
FILE_PATH="$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')"
# BAD — shell injection risk
FILE_PATH=$(echo $INPUT | jq -r .tool_input.file_path)
```

### Never log secrets

Filter out `.env` contents, API keys, tokens, and passwords from log files. Log the action, not the sensitive content.

### Permission layering — the precedence order

User-set `permissions.deny` rules always win over hook `permissionDecision: "allow"`. This is the **most important security invariant** of the hook system: a plugin's allowlist hook can streamline approvals for known-safe operations, but it cannot override what the user has explicitly forbidden. The official docs confirm:

> The permission layer evaluates rules per-subcommand. Explicit user `permissions.deny` rules take precedence over hook decisions.

**Practical consequence**: when writing an allowlist hook, document this contract in the script docstring. Users reviewing your plugin must be able to verify that your hook cannot escape their deny rules.

### Hook safety reminders from the official docs

- Hooks run with **your system user's full permissions** — they can access anything your account can.
- Use absolute paths or `$CLAUDE_PROJECT_DIR` for any file reference.
- Wrap shell profile `echo` statements in `if [[ $- == *i* ]]; then ... fi` — unconditional echo in `.bashrc` breaks JSON parsing in hooks.
- Admins can set `allowManagedHooksOnly: true` in managed policy to disable all user/project/plugin hooks org-wide.

---

## 5. Performance Best Practices

### Use async for non-blocking work

If the hook doesn't need to block the agent, use `"async": true`:

```json
{
  "type": "command",
  "command": "python .claude/hooks/log-action.py",
  "async": true
}
```

Async hooks run in background and deliver output on the next conversation turn. They **cannot block** a tool call — even if they exit 2, the call has already proceeded.

### Set appropriate timeouts

| Hook purpose | Recommended timeout |
|-------------|-------------------|
| Logging (simple append) | 10 s |
| File formatting (Prettier) | 30 s |
| Lint / type checks | 60 s |
| Test suite execution | 300 s |
| Deployment hooks | 600 s (default max) |

### Keep command hooks light

- Prefer `jq` piped to `>>` for simple logging over Python scripts.
- Use Python / Node only when you need complex logic.
- Avoid network calls in synchronous hooks unless necessary — they block the agent loop.

### Deduplicate naturally

Claude Code automatically deduplicates identical hook commands. Don't add your own dedup logic — it's handled for you.

### Hooks run in parallel

All matching hooks for a single event fire **in parallel**, not sequentially. Don't write hooks that depend on execution order — each hook should be self-contained.

### Hook input enrichment over time

- v2.1.119: hooks receive a `duration_ms` field.
- v2.1.121: PostToolUse hooks can replace tool output via `updatedToolOutput`.
- v2.1.133: hooks receive `effort.level` and `$CLAUDE_EFFORT` env var.
- v2.1.139: hooks support exec-form `args: string[]` (no shell-escaping).
- v2.1.141: hooks can emit `terminalSequence` for ANSI / OSC / window-title / bell sequences.
- v2.1.145: Stop / SubagentStop hooks receive `background_tasks` and `session_crons`.

When designing new hooks, take advantage of these inputs — `duration_ms` enables performance dashboards without instrumentation; `background_tasks` lets a Stop hook check "did any spawned task fail before allowing completion?"

---

## 6. Hook Event Selection Guide

The 29 events split into broad categories. Pick the event that fires *exactly* when you want the action to happen — earlier means more leverage to block, later means more information to react to.

### Quick decision tree

```
Want to block something Claude is about to do?
  → PreToolUse (per tool call)
  → PermissionRequest (per permission dialog)
  → UserPromptSubmit (per user prompt — rarely the right choice)
  → Stop (block completion — quality gates)
  → TaskCompleted (block task marked done)

Want to react to something Claude just did?
  → PostToolUse (per successful tool call)
  → PostToolUseFailure (per failed tool call)
  → PostToolBatch (after a parallel batch resolves)
  → SubagentStop (after a subagent finishes)
  → Stop (after Claude finishes responding)

Want to inject context?
  → SessionStart (at session boot, source: startup/resume/compact)
  → SubagentStart (at subagent spawn — specialist routing)
  → InstructionsLoaded (when CLAUDE.md or rules file loads)
  → PreCompact (before compaction wipes context)

Want to track environment?
  → ConfigChange (settings file changed mid-session)
  → CwdChanged (working directory changed)
  → FileChanged (watched file modified on disk)
  → WorktreeCreate / WorktreeRemove (worktree lifecycle)

Want telemetry?
  → SessionStart, SubagentStart, SubagentStop, Stop, SessionEnd
    (all five together = full session lifecycle log)

Want to handle MCP elicitation?
  → Elicitation (MCP server requests user input)
  → ElicitationResult (user response captured)
```

### When NOT to use each event

| Event | Don't use when |
|-------|---------------|
| `UserPromptSubmit` | You want to filter outputs — use Stop or PostToolUse instead |
| `PreToolUse` | You want to *modify* tool output — that's PostToolUse with `updatedToolOutput` |
| `PostToolUse` | You want to block — too late, tool already ran |
| `SessionStart: startup` | You want to inject context after compaction — also bind `compact` |
| `Stop` | You don't check `stop_hook_active` — infinite loops if you skip the check |
| `WorktreeRemove` | Agent writes untracked files — see section 2.7 |
| `Notification` | You want to log everything — too narrow; use PostToolUse |

### Choosing between SessionStart matchers

| Matcher | Fires when | Typical use |
|---------|-----------|-------------|
| `startup` | First boot of a session | Long-lived context injection — project conventions |
| `resume` | `claude --resume` reopens a session | Re-state rules that may have been compacted out |
| `clear` | `/clear` issued | Reset visible state, not config |
| `compact` | Context compaction completes | Re-inject the rules that just got wiped |

Bind multiple matchers in the same event:

```json
{
  "SessionStart": [
    { "matcher": "startup", "hooks": [...] },
    { "matcher": "compact", "hooks": [...] }
  ]
}
```

Or use empty `"matcher": ""` to fire on every source (idempotent hooks only).

### Choosing between PreToolUse and PermissionRequest

- **PreToolUse** fires for every tool call, regardless of permission state. Use it for blanket validation (block dangerous Bash, validate file paths).
- **PermissionRequest** fires only when the permission dialog would otherwise appear. Use it to *replace* the dialog with a programmatic decision.

Most plugin allowlists belong in `PreToolUse` — they short-circuit the permission flow entirely with `permissionDecision: "allow"`.

---

## 7. Stop Hook Best Practices

### Always check `stop_hook_active`

Prevent infinite loops where your Stop hook causes Claude to continue, which triggers another Stop:

```bash
INPUT=$(cat)
STOP_ACTIVE=$(echo "$INPUT" | jq -r '.stop_hook_active // false')

if [ "$STOP_ACTIVE" = "true" ]; then
  exit 0  # Already in a stop-hook cycle, let it finish
fi

# Your actual stop logic here
```

### Use Stop hooks for quality gates

The **why:** Stop hooks are the last point before Claude declares completion. Running tests / linters / typecheckers here turns "the agent said it's done" into "the agent's work passes the gate before being declared done." The block JSON (`{"decision":"block","reason":"..."}`) returns control to Claude with the failure context — it gets to fix and re-stop.

Always check `stop_hook_active` first to prevent infinite re-entry loops.

> **Mechanics:** see `how-to-create-hooks.md` § 6.5 for the canonical Bash recipe with the loop guard.

John Lindquist's pattern (from the *Advanced Claude Code techniques* episode of Lenny's Podcast) chains four checks in a Stop hook: (1) detect file changes, (2) run quality checks (lint, type-check, tests), (3) handle errors with a block decision, (4) commit on success.

> "Stop hooks chain check changes → quality checks → handle errors → commit on success." — John Lindquist, *Advanced Claude Code techniques* on Lenny's Podcast — https://www.lennysnewsletter.com/p/advanced-claude-code-techniques-context

The TypeScript validation case — fail the run if `tsc` errors remain — is structurally identical to the dbt plugin's `validate-dbt-structure.py` hook.

### Use SubagentStop for per-agent gates

The same pattern works for subagents — block completion until their output passes validation:

```bash
INPUT=$(cat)
TRANSCRIPT=$(echo "$INPUT" | jq -r '.agent_transcript_path')
# Validate the subagent's output
python validate_subagent_output.py "$TRANSCRIPT" || \
  echo '{"decision":"block","reason":"Subagent output failed validation"}'
```

---

## 8. Subagent Monitoring Best Practices

### Track subagent lifecycle

Log both start and stop events to understand subagent behavior:

```json
{
  "hooks": {
    "SubagentStart": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "python .claude/hooks/log-subagent.py",
        "statusMessage": "Tracking subagent..."
      }]
    }],
    "SubagentStop": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "python .claude/hooks/log-subagent.py",
        "statusMessage": "Recording subagent result..."
      }]
    }]
  }
}
```

### Inject context into subagents

The **why:** `SubagentStart` hooks fire just before the subagent's first turn, with the agent name available as `matcher`. This is the cleanest place to push project- or agent-specific framing without bloating the agent's static system prompt — particularly useful when the same agent type (`Explore`, `general-purpose`) is spawned from different code paths that each want different focus areas.

> **Mechanics:** see `how-to-create-hooks.md` § 6.7 for the canonical JSON registration (matcher = agent name, command = echo of the framing).

### Subagent observability — the multi-agent dashboard pattern

For multi-agent orchestration, ship all four lifecycle events (`SubagentStart`, `SubagentStop`, `Stop`, `SessionEnd`) to a shared stream. IndyDevDan's `claude-code-hooks-multi-agent-observability` reference architecture: `Claude Agents → Hook Scripts → HTTP POST → Bun Server → SQLite → WebSocket → Vue Client`. The point isn't the specific stack — it's that hook events are the only place to capture multi-agent activity *without* modifying the agents themselves.

The official Anthropic answer to this pattern arrived as the in-CLI Agent View / dashboard UI (covered in Sam Witteveen's "Claude Code Just Got an Agent Dashboard"). For your own plugins, write JSONL to disk and let users wire a dashboard up if they want one.

---

## 9. Session Management Best Practices

### Inject context on startup and after compaction

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [{
          "type": "command",
          "command": "echo 'Project uses Git Bash on Windows. Use Unix commands only. Never auto-commit.'"
        }]
      },
      {
        "matcher": "compact",
        "hooks": [{
          "type": "command",
          "command": "cat \"$CLAUDE_PROJECT_DIR/.claude/context-reminder.md\""
        }]
      }
    ]
  }
}
```

The `compact` matcher is especially important: compaction wipes most of the working context, and any rule that hasn't been laminated into CLAUDE.md will be lost. Inject critical context (current task state, recent decisions, project invariants) on `compact` to keep the model anchored.

### Clean up on session end

```json
{
  "hooks": {
    "SessionEnd": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "python .claude/hooks/session-cleanup.py",
        "async": true
      }]
    }]
  }
}
```

`SessionEnd` is the right place for: log rotation, temp-file cleanup, daily-aggregate roll-ups, summary writeouts.

### Setup hook for one-time prep

The `Setup` event fires when Claude Code is started with `--init-only`, `--init`, or `--maintenance` in `-p` mode. Use it for CI / script preparation that should not run in interactive sessions:

```json
{
  "hooks": {
    "Setup": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/ci-bootstrap.py"
      }]
    }]
  }
}
```

---

## 10. Windows-Specific Best Practices

### The why (Windows is its own failure mode)

Hook commands on Windows **do not use Git Bash path translation**, even though Claude Code itself requires Git Bash. This single mismatch is the #1 cause of "my hook is registered but does nothing" on Windows — and the failure is silent under `"async": true` (no error, no log, no status-line message). Path-format bugs there are indistinguishable from a hook that simply doesn't fire.

The mitigation is mechanical: **always use Windows-native forward-slash paths (`C:/Python313/python.exe`), never `/c/...` mount paths, never `~`, never `$HOME`.** Async hooks should remain async only after the path is proven working with a synchronous test run.

Two additional Windows-only failure modes worth keeping in scope:

- **Shell profile pollution.** Unconditional `echo` in `.bashrc` / `.zshrc` breaks JSON parsing in any hook that reads stdin. Guard with `if [[ $- == *i* ]]; then ... fi`.
- **Notifications.** No native cross-platform path — shell out to PowerShell's `System.Windows.Forms.MessageBox`.

> **Mechanics:** see `how-to-create-hooks.md` § 8 ("Cross-Platform Considerations") for:
> - The four-row Windows path-format table (works / fails-silently breakdown)
> - A working Windows JSON registration example
> - The synchronous-hook → test-beacon debugging strategy
> - Linux / macOS parity notes

---

## 11. Testing and Debugging Best Practices

### The why (hooks need their own tests)

Hooks live in the gap between the harness and your agent — they never run under the same conditions twice unless you build a fixture harness. Without one, the only way to find out a hook regression broke is to notice the agent doing something it shouldn't (or failing silently to do something it should). Fixture-driven testing closes that gap: one JSON file per event you care about, a small runner that pipes each to its hook and asserts exit code + stdout, run from CI on every hook change.

Three diagnostics worth knowing about for in-flight debugging:

- **`claude --debug`** — surfaces hook execution details inline.
- **`Ctrl+O` (verbose mode)** — toggles per-session verbose output including hook progress and non-blocking stderr.
- **`/hooks`** — lists every active hook with its source scope (`[User]`, `[Project]`, `[Local]`, `[Plugin: foo]`); the fastest way to confirm a new hook is registered and not shadowed.

> **Mechanics:** see `how-to-create-hooks.md` § 7 ("Testing Hooks") for:
> - The manual `echo JSON | python hook.py` pattern
> - A complete fixtures folder layout
> - A Python `test_hooks.py` runner (subprocess + assertion pattern, drop-in CI-ready)
> - The debug-mode / verbose-mode / `/hooks` triad detailed

### Defensive exit codes — Baugues's "always exit 0" rule

When a hook is observational (logging, telemetry, notifications) rather than gating, the defensive default is to never let its own failure interrupt the agent. Exit 0 unconditionally.

> "Always exit with code 0 (success) so we don't interrupt Claude's work, even if something went wrong." — Greg Baugues, *Claude Code: Getting Started with Hooks* (https://www.baugues.com/hooks/ — companion blog to https://www.youtube.com/watch?v=8T0kFSseB58).

This applies to PostToolUse loggers, Notification scripts, SubagentStart context-injectors, and any hook whose role is "observe, don't decide." Reserve exit codes 1 and 2 (and the `block` JSON response) for hooks whose explicit job is to gate or veto.

### Pre-shipment audit gates (plugin authors)

As section 2.11 details, ship an audit script in `tests/preshipment_audit.py` that validates the plugin's hook configuration before each release. Run it in CI; block PRs that fail. Cheap, catches drift early.

---

## 12. Anti-Patterns to Avoid

| Anti-pattern | Why it's bad | Do this instead |
|-------------|-------------|-----------------|
| Inline complex bash in JSON | Hard to read, debug, maintain | Use external script files |
| Not checking `stop_hook_active` | Infinite loop | Always check and exit 0 if true |
| Synchronous network calls | Blocks agent for seconds | Use `"async": true` |
| Logging sensitive data | Security risk | Log actions, not content |
| Modifying files in PreToolUse hooks | Side effects before tool runs | Use PostToolUse instead |
| Not quoting variables | Shell injection | Always quote: `"$VAR"` |
| Hooks that depend on execution order | Hooks run in parallel | Each hook should be self-contained |
| Over-hooking everything | Performance overhead, noise | Start minimal, add as needed |
| Not using `jq` | Fragile JSON parsing with grep / sed | Install `jq`, use it everywhere |
| Writing hooks without testing | Breaks agent workflow | Test with piped JSON first |
| Using `~`, `$HOME`, or `/c/` paths on Windows | Hooks fail silently | Use `C:/` Windows-native paths |
| Hardcoding `python3` in plugin hooks | Breaks on Windows (which has `python.exe`) | Use `python` — the launcher picks the version |
| Loading allowlists from external config | Cannot be audited at install time | Inline the patterns in the hook script |
| Exit 2 with no stderr message | User sees a block with no explanation | Always write a message to stderr first |
| Allowing `permissionDecision: "deny"` in plugin allowlist hooks | Surprises users — overrides their permissions | Allowlist hooks should ALLOW only; return `{}` for unknown patterns |
| `isolation: worktree` + agent writes untracked files | WorktreeRemove wipes the worktree silently | See section 2.7 — drop worktree or commit-and-merge-back |
| Embedding agent-specific `hooks:` in plugin agent.md frontmatter | Silently stripped at install time | Put hooks in `plugin.json` |

---

## 13. External Source Validation (Citations)

Compact table of load-bearing claims with their citation.

| Claim | Source | Quote |
|-------|--------|-------|
| Hooks are deterministic, CLAUDE.md is advisory | `code.claude.com/docs/en/best-practices` | *"Hooks run scripts automatically at specific points in Claude's workflow. Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens."* |
| Use hooks for "must happen every time" actions | `code.claude.com/docs/en/best-practices` | *"Use hooks for actions that must happen every time with zero exceptions."* |
| Claude can write hooks for you | `code.claude.com/docs/en/best-practices` | *"Try prompts like 'Write a hook that runs eslint after every file edit' or 'Write a hook that blocks writes to the migrations folder.'"* |
| Plugins bundle hooks as one of 4 payload types | `claude.com/blog/claude-code-plugins` | *"Plugins are a lightweight way to package and share any combination of: Slash commands, Subagents, MCP servers, and Hooks."* |
| Plugins help standardize team workflows via hooks | `claude.com/blog/claude-code-plugins` | *"Engineering leaders can maintain consistency across their team by using plugins to ensure specific hooks run for code reviews or testing workflows."* |
| Hooks are part of the autonomy bundle | `anthropic.com/news/enabling-claude-code-to-work-more-autonomously` (2025-09-29) | *"Hooks automatically trigger actions at specific points, such as running your test suite after code changes or linting before commits."* |
| Hooks always exit 0 defensively (community wisdom) | `baugues.com/hooks` (Greg Baugues) | *"Always exit with code 0 (success) so we don't interrupt Claude's work, even if something went wrong."* |
| Hooks are defense-in-depth + observability layer | IndyDevDan, "I'm HOOKED on Claude Code Hooks" (`youtube.com/watch?v=J5B9UGTuNoM`) | *"As we push into the age of agents, we need observability to scale our impact. We need to know what our agents are doing."* |
| Plugin-shipped agents cannot declare `hooks:` in frontmatter | `code.claude.com/docs/en/plugins-reference` | *"For security reasons, `hooks`, `mcpServers`, and `permissionMode` are not supported for plugin-shipped agents."* |
| `${CLAUDE_PLUGIN_ROOT}` is the canonical plugin path | `code.claude.com/docs/en/plugins-reference` | *"Each value is available for substitution as `${user_config.KEY}` in MCP and LSP server configs, hook commands, and monitor commands."* |
| `CLAUDE_PLUGIN_ROOT` not set for SessionStart hooks | GitHub Issue #27145 | Open as of v2.1.150 |
| `CLAUDE_PLUGIN_ROOT` not passed to UserPromptSubmit hooks | GitHub Issue #36585 | Open as of v2.1.150 |
| `CLAUDE_PLUGIN_ROOT` inconsistent for local marketplace plugins | GitHub Issue #38699 | *"hooks receive the source directory while the agent's shell environment receives the cache/install directory"* |
| Hook handler types | `code.claude.com/docs/en/plugins-reference` | "Hook handler types: `command`, `http`, `mcp_tool`, `prompt`, `agent`." |
| Worktree hooks tied to `isolation: 'worktree'` agent field | `code.claude.com/docs/en/plugins-reference` | *"The only valid `isolation` value is 'worktree'."* |
| Stop / SubagentStop receive `background_tasks` / `session_crons` | Changelog v2.1.145 | *"Stop/SubagentStop hook input includes `background_tasks` and `session_crons`."* |
| Hooks receive `effort.level` and `$CLAUDE_EFFORT` | Changelog v2.1.133 | *"Hooks receive `effort.level` and `$CLAUDE_EFFORT`."* |
| Hooks support exec-form `args` field | Changelog v2.1.139 | *"Added hook `args: string[]` field (exec form)."* |
| PostToolUse can replace tool output via `updatedToolOutput` | Changelog v2.1.121 | *"PostToolUse hooks replace output via `updatedToolOutput`."* |
| Hooks emit ANSI / OSC via `terminalSequence` | Changelog v2.1.141 | *"Added `terminalSequence` field to hook JSON output for desktop notifications, window titles, bells."* |
| MCP tools callable from hooks via `type: 'mcp_tool'` | Changelog v2.1.118 | *"Hooks invoke MCP tools via `type: 'mcp_tool'`."* |
| SessionStart hooks get fresh env vars per session | Changelog v2.1.136 | *"Fixed SessionStart hooks env vars stale."* |
| Hooks receive `duration_ms` field | Changelog v2.1.119 | *"Hooks receive `duration_ms` field."* |
| Hook `if` conditions fixed (e.g. `PowerShell(git push*)`) | Changelog v2.1.147 | *"Fixed hook `if` conditions like `PowerShell(git push*)` never matching."* |
| Prompt / agent hooks now give clearer errors for SessionStart binding | Changelog v2.1.142 | *"Improved hook configuration error for prompt/agent-type hooks on `SessionStart`/`Setup`/`SubagentStart`."* |
| Plan-mode subagents bypass plan-mode constraint | GitHub Issue #43777 | *"using the Agent tool spawns subagents that are not subject to the plan mode constraint"* |
| Hooks documented at official URL | `code.claude.com/docs/en/hooks` and `/hooks-guide` | Primary reference |
| Stop hooks for TS validation (community pattern) | John Lindquist on Lenny's Podcast (`lennysnewsletter.com/p/advanced-claude-code-techniques-context`) | Stop hooks chain check changes → quality checks → handle errors → commit on success |

---

## 14. Further Reading

### Companion docs in this folder

- `how-to-create-hooks.md` — step-by-step mechanics (29-event catalogue, JSON contract, worked examples, troubleshooting)
- `claude-code-plugin-best-practices.md` — broader plugin packaging
- `how-to-create-plugins.md` — plugin step-by-step guide
- `claude-code-agent-best-practices.md` and `claude-code-skill-best-practices.md` — sibling best-practices docs
- All external citations (Anthropic release notes, blog posts, expert YouTube videos) are embedded inline as `> blockquote` citations in § 13.

### Official Anthropic docs

- `code.claude.com/docs/en/hooks` — hooks reference
- `code.claude.com/docs/en/hooks-guide` — hooks guide
- `code.claude.com/docs/en/plugins-reference` — plugin schema (includes hooks section)
- `code.claude.com/docs/en/best-practices` — best-practices document (includes hooks section)
- `claude.com/blog/how-to-configure-hooks` — original how-to blog post
- `anthropic.com/news/enabling-claude-code-to-work-more-autonomously` — autonomy update (hooks + subagents + background + checkpoints)
- `claude.com/blog/claude-code-plugins` — plugins launch post (lists hooks as one of four plugin payloads)

### Community reference implementations and tutorials

- `github.com/disler/claude-code-hooks-mastery` — IndyDevDan's reference repo
- `github.com/disler/claude-code-hooks-multi-agent-observability` — hooks → WebSocket → dashboard
- `github.com/disler/claude-code-damage-control` — defense-in-depth PreToolUse blocks
- `baugues.com/hooks` — Greg Baugues's getting-started tutorial (single-file Python handler with full code)
- `dev.to/lukaszfryc/claude-code-hooks-complete-guide-with-20-ready-to-use-examples-2026-dcg` — 20+ hook recipes

### Key videos

- IndyDevDan, "I'm HOOKED on Claude Code Hooks: Advanced Agentic Coding" — `youtube.com/watch?v=J5B9UGTuNoM`
- IndyDevDan, "How I Use Claude Code Hooks to Power My Indie Dev Life" — `youtube.com/watch?v=RBtCwuKCKdE`
- IndyDevDan, "Claude Code Hooks - Deep Mastery" playlist — `youtube.com/playlist?list=PLS_o2ayVCKvBR3jawG9JFIzJ1vXffi8fS`
- Greg Baugues, "Claude Code - Getting Started with Hooks" — `youtube.com/watch?v=8T0kFSseB58`
- John Lindquist on Lenny's Podcast, "Advanced Claude Code techniques" — `lennysnewsletter.com/p/advanced-claude-code-techniques-context`
- "How Hooks Are Transforming AI-Driven Development" — `youtube.com/watch?v=LkFV6JESci8`

### Quick start checklist

1. Install `jq` if not already available.
2. Create `.claude/hooks/` directory in your project.
3. Start with **SessionStart** (context injection) and **Notification** (desktop alerts).
4. Add **PreToolUse** for safety (block dangerous commands / files).
5. Add **PostToolUse** for logging (track what the agent did).
6. Add **SubagentStart / Stop** for subagent observability.
7. Add **Stop** hook for quality gates (run tests before completing).
8. Test each hook manually with piped JSON.
9. Review with `/hooks` command.
10. Debug with `claude --debug` or `Ctrl+O` verbose mode.
