# How to Create Claude Code Plugins

**Last Updated:** 2026-05-25
**Purpose:** Build, package, test, and ship Claude Code plugins that survive a fresh install.

This guide is one of four how-tos alongside `how-to-create-agents.md`, `how-to-create-skills.md`, and `how-to-create-hooks.md`. It captures the empirical lessons from shipping two production plugins — `dbt-pipeline-toolkit` (9 agents, 8 skills, 3 hooks, 1 MCP server) and `fabric-dataflow-migration-toolkit` (6 agents, 8 skills, 3 hooks). Everything in it is the result of things that **actually broke** when a plugin was installed on a second machine — not theory.

---

## Table of Contents

1. [What is a Plugin?](#1-what-is-a-plugin)
2. [Plugin vs Agent vs Skill — When to Use Which](#2-plugin-vs-agent-vs-skill--when-to-use-which)
3. [Plugin Repository Structure](#3-plugin-repository-structure)
4. [The plugin.json Manifest](#4-the-pluginjson-manifest)
5. [userConfig — Install-Time Configuration](#5-userconfig--install-time-configuration)
6. [Bundling Agents into a Plugin](#6-bundling-agents-into-a-plugin)
7. [Bundling Skills into a Plugin](#7-bundling-skills-into-a-plugin)
8. [Bundling Hooks into a Plugin](#8-bundling-hooks-into-a-plugin)
9. [Bundling an MCP Server into a Plugin (Optional)](#9-bundling-an-mcp-server-into-a-plugin-optional)
10. [Reference Materials Inside a Plugin](#10-reference-materials-inside-a-plugin)
11. [The Marketplace (Distribution)](#11-the-marketplace-distribution)
12. [Installation Walkthrough](#12-installation-walkthrough)
13. [Testing — Fresh Install Is Non-Negotiable](#13-testing--fresh-install-is-non-negotiable)
14. [Pre-Shipment Audit](#14-pre-shipment-audit)
15. [Versioning & Release Process](#15-versioning--release-process)
16. [Common Pitfalls](#16-common-pitfalls)
17. [Companion Documents](#17-companion-documents)

---

## 1. What is a Plugin?

A Claude Code plugin is a single installable unit that bundles agents, skills, hooks, MCP servers, and reference content under a versioned manifest, distributable through a marketplace.

You ship one repo, declare its contents in `.claude-plugin/plugin.json`, and your users add the marketplace once and `/plugin install <name>` whenever they want it. Updates, uninstalls, and configuration prompts are managed for you by Claude Code.

**Why bundle a plugin instead of distributing loose agents and skills:**

- **Ergonomic install** — one `/plugin install` command instead of `cp -r` into `~/.claude/agents/` plus `~/.claude/skills/` plus hand-editing `settings.json` for hooks.
- **Single security surface** — every privilege the plugin escalates (hooks, MCP servers, allowlisted Bash) is declared in one auditable file (`plugin.json`).
- **Versioned** — `version` field in `plugin.json` plus a marketplace registry means users can pin, upgrade, or roll back.
- **Distributable** — marketplaces (yours or third-party) provide discovery. Users `/plugin marketplace add <repo>` once and see every plugin you publish there.
- **Lifecycle management** — Claude Code copies the plugin to a cache directory, garbage-collects orphan versions after 7 days, and isolates plugin code from user-edits.

---

## 2. Plugin vs Agent vs Skill — When to Use Which

| Mechanism | Lives Where | Versioned | Shipped How | When to Use |
|---|---|---|---|---|
| **Single agent file** | `~/.claude/agents/<name>.md` | No | Manual copy | One role, one user, no install ceremony |
| **Single skill folder** | `~/.claude/skills/<name>/` | No | Manual copy | One capability, one user, content stable |
| **Plugin** | `~/.claude/plugins/cache/<id>/` | Yes (semver) | Marketplace install | Bundling multiple agents/skills + cross-machine repeatability + scoped privileges (hooks, MCP) + multi-machine deployment |

**Rules of thumb:**

- If you find yourself writing setup instructions like "now copy `agent.md` into `~/.claude/agents/`, then run `pip install ...`, then add this hook to `~/.claude/settings.json`," you should be shipping a plugin.
- If your work is a single agent file that you tweak weekly for your own use, keep it standalone — plugins have a versioning ceremony that slows you down for personal iteration.
- If your work uses a hook, an MCP server, or any privileged setting that lives in `settings.json`, it must be a plugin — that's the only way to ship those things as part of a single auditable artifact.
- If you want multiple users to install the same thing reliably, you must ship a plugin. Loose-file distribution invariably ends in "works on my machine" failures (see Section 13).

---

## 3. Plugin Repository Structure

Standard layout (mirrors what both shipped plugins use):

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json              # REQUIRED — the manifest
├── agents/
│   ├── orchestrator/            # subdirectory layout (3-part namespace)
│   │   └── agent.md
│   └── specialist-1/
│       └── agent.md
├── skills/
│   ├── data-runner/
│   │   ├── SKILL.md             # REQUIRED for the skill
│   │   └── scripts/
│   │       └── run.py
│   └── source-reader/
│       ├── SKILL.md
│       └── scripts/
├── hooks/
│   ├── approve-plugin-bash.py   # PreToolUse Bash allowlist
│   ├── validate-structure.py    # PreToolUse Write/Edit validator
│   └── session-start-check.py   # SessionStart config check
├── servers/                     # optional MCP server
│   ├── package.json
│   ├── src/...
│   └── dist/...                 # ship compiled output
├── reference/                   # docs the agents read
│   ├── style-guide.md
│   └── examples/
├── examples/                    # bundled sample inputs (for --sample mode)
├── tests/
│   └── preshipment_audit.py
├── _Documentation/              # design docs, plugin_learnings.md
├── _Research/                   # research artifacts
├── _Plan/                       # plans + Backlog.md + Issues.md
├── CLAUDE.md                    # for plugin contributors
└── README.md                    # for plugin users
```

**Key points:**

- The `.claude-plugin/` folder is the canonical home for `plugin.json` — that's the path Claude Code looks for when it loads a plugin. It is **never** `plugin.json` at repo root, and the marketplace's `marketplace.json` does **not** live alongside `plugin.json` in the same repo (see Section 11).
- The plugin's own `CLAUDE.md` documents the plugin **for contributors** (the people building / patching the plugin). `README.md` documents it **for users** (the people installing it). Both belong at repo root.
- Folder names `agents/`, `skills/`, `hooks/`, `servers/`, `reference/` are by convention — Claude Code discovers contents by what's referenced in `plugin.json` and by what it finds inside `agents/` and `skills/`, so the convention matters more than any hard rule.
- `_Plan/Issues.md` and `_Plan/Backlog.md` aren't required by Claude Code, but every release worth shipping needs an issue tracker — these two files keep undocumented fixes from accumulating (Source: DBT plugin's `_Plan/` policy).

---

## 4. The plugin.json Manifest

This file is the auditable surface of your plugin. Every privilege escalation, every MCP server, every hook, every userConfig field is declared here. If something isn't in `plugin.json`, Claude Code doesn't see it.

### Field reference

```jsonc
{
  "name": "my-plugin",                   // REQUIRED — kebab-case, lowercase
  "version": "1.0.0",                    // REQUIRED — semver
  "description": "What it does.",        // REQUIRED — one sentence
  "author": {
    "name": "Author Name",
    "email": "name@example.com",
    "url": "https://github.com/handle"
  },
  "homepage": "https://github.com/handle/my-plugin",
  "repository": "https://github.com/handle/my-plugin",
  "license": "MIT",
  "keywords": ["data", "automation"],

  "mcpServers": { /* see Section 9 */ },
  "hooks": { /* see Section 8 */ },
  "userConfig": { /* see Section 5 */ }
}
```

### Real example (excerpted)

From `dbt-pipeline-toolkit`'s manifest, showing how an MCP server, hooks, and userConfig all wire together through `${user_config.*}` and `${CLAUDE_PLUGIN_ROOT}`:

```json
{
  "name": "dbt-pipeline-toolkit",
  "version": "1.0.0",
  "mcpServers": {
    "sql-server-mcp": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/servers/dist/minimal-mcp-server.js"],
      "env": {
        "SQL_SERVER": "${user_config.sql_server}",
        "SQL_PASSWORD": "${user_config.sql_password}"
      }
    }
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command",
            "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/validate-dbt-structure.py" }
        ]
      }
    ]
  }
}
```

### Two substitution variables you must know

- **`${CLAUDE_PLUGIN_ROOT}`** is substituted at load for every path reference inside the manifest, in agent/skill markdown body content, and in hook commands. It resolves to the plugin's installed cache path on a fresh install (`~/.claude/plugins/cache/<id>/`) and to the repo root in dev mode (`--plugin-dir ./`). Use it **everywhere** that references a file bundled with the plugin. NEVER hardcode `$HOME/.claude/skills/...` or any user-home-relative path — that path doesn't exist on a fresh install (Source: DBT F7).
- **`${user_config.<key>}`** is substituted inside `mcpServers.<name>.env`, inside `hooks.*.command`, and inside skill/agent body content (where templated). It pulls from the values your user supplied at install time via the `userConfig` schema.

### The manifest is your security contract

A plugin-level hook or MCP server declared in `plugin.json` is greppable, reviewable, and surfaced in `/plugin` inspection commands. That's why Claude Code strips elevated-privilege fields (`hooks`, `mcpServers`, `permissionMode`) from individual agent frontmatter (see Section 6) — those would be runtime-only declarations not visible at install time. The asymmetry is deliberate: privileges concentrate in the manifest so users can audit before enabling (Source: DBT F2).

---

## 5. userConfig — Install-Time Configuration

`userConfig` declares the values Claude Code prompts your user for when they install the plugin. It is the official alternative to making users hand-edit `settings.json`.

### Schema

```json
"userConfig": {
  "sql_server": {
    "title": "SQL Server Hostname",
    "description": "SQL Server hostname (e.g., localhost). Leave empty to set at runtime.",
    "type": "string",
    "sensitive": false
  },
  "sql_password": {
    "title": "SQL Password",
    "description": "SQL Server password. Stored in system keychain.",
    "type": "string",
    "sensitive": true
  }
}
```

**Fields:**
- `title` — short label for the prompt
- `description` — longer explanation surfaced beneath the prompt
- `type` — usually `"string"`
- `sensitive` — if `true`, the value is written to your system keychain (Windows Credential Manager / macOS Keychain / libsecret) rather than to `settings.json`

### Where values land

- **Non-sensitive** values are persisted to `~/.claude/settings.json` under `pluginConfigs.<plugin-name>.options.<key>`.
- **Sensitive** values are written to the OS keychain under a Claude-Code-controlled service name.
- **Both** are surfaced inside the manifest as `${user_config.<key>}` for template substitution.
- **Both** are exported to every plugin subprocess as `CLAUDE_PLUGIN_OPTION_<KEY>` (uppercased).

### The env-var remap trap (Source: DBT F5 Problem A)

The env var passed to subprocesses is `CLAUDE_PLUGIN_OPTION_<KEY>`, **not** the bare `<KEY>`. Your skill scripts that call `os.environ.get('SQL_SERVER')` will see nothing on a fresh install — the value is in `CLAUDE_PLUGIN_OPTION_SQL_SERVER`.

There are two fixes:

1. **Explicit env mapping in `plugin.json`** for MCP servers — see the `mcpServers.<name>.env` block above. Works because Claude Code substitutes `${user_config.sql_server}` *before* spawning the server, so the server starts with bare `SQL_SERVER` set.

2. **`_load_plugin_userconfig_env()` helper for every Python skill script** — runs at module load time, before `argparse` evaluates defaults, and copies `CLAUDE_PLUGIN_OPTION_<KEY>` → `<KEY>`:

```python
def _load_plugin_userconfig_env():
    keys = (
        'SQL_SERVER', 'SQL_DATABASE', 'SQL_AUTH_TYPE', 'SQL_USER', 'SQL_PASSWORD',
        'AZURE_TENANT_ID', 'AZURE_CLIENT_ID', 'AZURE_CLIENT_SECRET',
    )
    for key in keys:
        if not os.environ.get(key):
            fallback = os.environ.get(f'CLAUDE_PLUGIN_OPTION_{key}')
            if fallback:
                os.environ[key] = fallback

_load_plugin_userconfig_env()  # MUST run before argparse default evaluation
```

Duplicate this helper into every script that needs userConfig env vars. Sharing it through a single connection module won't work if consumer scripts lazy-import the module *after* their own `argparse` has already evaluated defaults from `os.environ.get(...)`.

### The install-time-prompt may silently skip (Source: DBT F5 Problem B)

Even with a well-formed `userConfig` block, the interactive prompt **may not fire** on some install or update flows. Mechanism unknown. Defensive measures:

1. Ship a **SessionStart hook** (`hooks/session-start-config-check.py`) that detects missing userConfig env vars and prints a "Setup Required" message with exact `settings.json` paths the user can hand-edit (Source: Fabric plugin manifest, which ships `session-start-config-check.py` for exactly this reason).
2. Document the manual config paths prominently in `README.md` — `~/.claude/settings.json` → `pluginConfigs.<plugin-name>.options.<key>`.
3. For sensitive values that didn't prompt, point users at the keychain entry your plugin would write to.

---

## 5.5. Atomic Bash (Quick Stub)

Every Bash command surfaced by a plugin — inside any agent body, SKILL.md, or hook command — **must be a single atomic operation**. No `&&`, `||`, `;`, `|`, subshells `(...)`, `$(...)`, backticks, or heredocs.

**Why:** Claude Code's permission layer matches rules per-subcommand. Compound expressions fall through to interactive prompts, which block background subagents silently (no channel to answer the prompt → tool call stalls). The pre-shipment audit's `atomic_bash` gate (§ 14) enforces this mechanically; the pitfall table (§ 16) catalogues every regression class.

**Full rationale + rewrite patterns:** see `claude-code-plugin-best-practices.md` § 5 ("Atomic Bash — No Compound Shell Expressions"). That section walks through the original DBT F9 incident, the cost analysis (1–3 % more tokens for a large reliability gain), and the rewrite patterns for "run A then B" / "run A; if fail run B" / "pipe A to B" / multi-step recipes.

---

## 6. Bundling Agents into a Plugin

Agents live under `agents/`. There are two layouts and they produce different namespaces.

### Flat vs subdirectory layout (Source: DBT F1)

| Layout | File path | Registered name on install | Namespace parts |
|---|---|---|---|
| **Flat** | `agents/security-reviewer.md` | `my-plugin:security-reviewer` | 2 |
| **Subdirectory** | `agents/orchestrator/agent.md` | `my-plugin:orchestrator:orchestrator` | 3 |

The docs example shows the flat case and implies 2-part is universal. It is not. The rule is:

> **Namespace segments = 1 (plugin name) + directory depth of the agent file relative to `agents/`.**

Flat files have depth 1 (the file itself). Subdirectory layouts (`agents/<name>/agent.md`) have depth 2 — the subdirectory plus the file. Both shipped plugins use the subdirectory layout because each agent has its own reference files / examples, and they all register as 3-part names.

**Hazard:** if the subdirectory name diverges from the frontmatter `name` field, the registered name silently follows the directory. Always keep them identical.

### Frontmatter restrictions (Source: DBT F2)

Plugin-shipped agents lose three fields that work fine in standalone agents:

| Field | Plugin-shipped agent | Standalone agent | Workaround |
|---|---|---|---|
| `permissionMode` | Silently stripped | Honored | Pass `mode: "acceptEdits"` at the call site in the parent's `Task(...)` call |
| `hooks` | Silently stripped | Honored | Declare at plugin level in `plugin.json` `hooks` block |
| `mcpServers` | Silently stripped | Honored | Declare at plugin level in `plugin.json` `mcpServers` block |

The stripping is silent — the agent loads and runs, the affected field is just gone. Background subagents that relied on stripped `permissionMode: acceptEdits` for write access will stall indefinitely on their first `Write` call (Source: DBT F3).

**Always do:**

```jsonc
// Parent agent calling a plugin-shipped specialist:
Task(
  subagent_type: "my-plugin:specialist:specialist",
  prompt: "...",
  run_in_background: true,
  mode: "acceptEdits"          // call-site permission, not stripped
)
```

### Cross-reference the full registered name everywhere

Every place that references another plugin agent — `tools: Agent(...)` allowlist, `subagent_type:` in `Task` calls, `claude --agent ...` examples in the README — needs the **full registered name**, with all its namespace segments. Bare names worked in dev (because `--plugin-dir ./` registers agents under bare names), and silently produce no-op spawns on install.

### Skills frontmatter uses 2-part namespace, not 3-part (Source: DBT F8)

An agent's `skills:` frontmatter (used to preload skills into subagent context) follows a *different* format:

```yaml
# 3-part for agent references (subagent_type, Agent(...)):
subagent_type: my-plugin:specialist:specialist

# 2-part for skill references (skills: frontmatter, Skill tool calls):
skills: my-plugin:data-runner, my-plugin:source-reader
```

The reason is that skills are always flat (depth 1) — they never get the extra namespace segment that subdirectory agents do. Bare skill names in the `skills:` field resolve to nothing in plugin-shipped agents, so the preload silently fails. Always use the 2-part form.

### Orchestrators must be launched as main-thread agents (Source: DBT F4, Fabric N11)

Claude Code enforces a one-level subagent hierarchy: the main thread can fan out to subagents, but those subagents are leaves — they cannot recursively spawn more subagents.

For your orchestrator to actually delegate, it must run as the main thread. The only invocation path that works is:

```bash
claude --agent my-plugin:orchestrator:orchestrator "<initial prompt>"
```

Specifically, **slash commands cannot host an orchestrator** that delegates to multiple subagents. A slash command runs inside an existing Claude session, so the orchestrator it spawns becomes a subagent itself, and the orchestrator's own `Task(...)` calls silently no-op. Either omit the slash command entirely or have it print "launch via `claude --agent ...` from a fresh shell."

---

## 7. Bundling Skills into a Plugin

Skills are flat — one folder per skill, no nested categories — and always register as 2-part names: `<plugin>:<skill>`.

```
skills/
├── data-runner/
│   ├── SKILL.md
│   └── scripts/
│       └── run.py
└── source-reader/
    ├── SKILL.md
    └── scripts/
        └── query.py
```

### Use `${CLAUDE_SKILL_DIR}` inside SKILL.md

Inside a SKILL.md, prefer `${CLAUDE_SKILL_DIR}/scripts/<file>.py` over `${CLAUDE_PLUGIN_ROOT}/skills/<name>/scripts/<file>.py`. `CLAUDE_SKILL_DIR` resolves to the skill's own directory and doesn't hardcode the skill's name inside its own instructions.

```bash
# Inside SKILL.md — works regardless of skill name:
python "${CLAUDE_SKILL_DIR}/scripts/profile_data.py" --file "$1"
```

Agent bodies, however, must use `${CLAUDE_PLUGIN_ROOT}/skills/<skill-name>/scripts/<file>.py` — `CLAUDE_SKILL_DIR` is scoped to the currently-active skill, which doesn't apply when the file being parsed is an agent definition.

### Plugin skills appear as "locked by plugin" (Source: DBT F8)

In the Claude Code UI, plugin skills are marked locked. This means:

1. The skill is owned by the plugin system — it lives in `~/.claude/plugins/cache/<id>/skills/<name>/`, not in the user's standalone `~/.claude/skills/`.
2. **The user cannot edit it in place.** Edits would be wiped on the next plugin update (Claude Code marks orphaned plugin versions for removal after 7 days).
3. Lifecycle is tied to the plugin's lifecycle — install adds, uninstall removes, `claude plugin update` replaces.
4. **No same-name override.** A user's personal `~/.claude/skills/data-runner/` and the plugin's `my-plugin:data-runner` coexist in different namespaces; neither overrides the other.
5. **Customization path is fork-the-plugin**, not edit-in-place. Same pattern as `node_modules/` or `site-packages/` — don't hand-edit, override at a higher level or fork upstream.

This raises the bar for plugin quality: users have no escape hatch to patch your bugs locally. Every release should be treated like a production deploy.

### Resolving bundled resources at runtime (Source: Fabric N16)

`CLAUDE_PLUGIN_ROOT` is set in the plugin/agent context but is **not guaranteed to propagate** into subprocesses launched by the orchestrator via the Bash tool. If a Python script relies on that env var to find its bundled templates / reference content, it will silently fall back to a wrong path when the var is missing.

The robust pattern: walk the script's own ancestors and validate by sentinel file:

```python
from pathlib import Path

SENTINEL = "risk-catalog.md"   # known file inside the reference folder

def resolve_reference_source():
    # 1. Try plugin-root env var first (when set)
    env_root = os.environ.get('CLAUDE_PLUGIN_ROOT')
    if env_root:
        candidate = Path(env_root) / "reference"
        if (candidate / SENTINEL).exists():
            return candidate

    # 2. Walk script ancestors and look for the sentinel
    for ancestor in Path(__file__).resolve().parents:
        candidate = ancestor / "reference"
        if (candidate / SENTINEL).exists():
            return candidate

    # 3. Loud failure with actionable remediation
    raise FileNotFoundError(
        "Could not locate plugin reference folder. "
        f"Looked for {SENTINEL} under CLAUDE_PLUGIN_ROOT and script ancestors."
    )
```

Rules of thumb:

1. Treat `CLAUDE_PLUGIN_ROOT` as a **hint**, not a contract. Walk ancestors as the reliable fallback.
2. Validate by **sentinel file presence**, not `dir.exists()` — a half-populated or stub folder will pass an existence check and ship broken content.
3. **Degrade loudly.** A "graceful degradation" branch that writes plausible-looking stub files is worse than a loud failure — downstream stages will hard-check for real content and fail far away from the actual problem.

---

## 8. Bundling Hooks into a Plugin

Hooks are at plugin level — registered in the `plugin.json` `hooks` block, never in agent frontmatter (which strips them; Source: DBT F2).

```json
"hooks": {
  "PreToolUse": [
    {
      "matcher": "Write|Edit",
      "hooks": [
        { "type": "command",
          "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/validate-structure.py",
          "statusMessage": "Validating file structure..." }
      ]
    }
  ]
}
```

### The three canonical hook patterns

Both shipped plugins use the same three hook patterns. Together they cover almost every plugin-shipping correctness concern.

#### Pattern A — PreToolUse Bash allowlist (Source: DBT F9)

**Problem:** Background subagents cannot satisfy permission prompts. `acceptEdits` mode auto-approves only filesystem moves (`mkdir`, `touch`, `mv`, `cp`) and file writes (`Write`/`Edit`) — **arbitrary Bash like `python profile_data.py` or `dbt run` still hits the normal permission flow**, which in background mode stalls silently.

**Fix:** Ship a plugin-level PreToolUse hook for `Bash` that auto-approves commands matching a narrow allowlist:

```json
{
  "matcher": "Bash",
  "hooks": [
    { "type": "command",
      "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/approve-plugin-bash.py",
      "statusMessage": "Checking plugin Bash allowlist..." }
  ]
}
```

The hook reads PreToolUse JSON from stdin, inspects the Bash command, and emits:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "Plugin-internal script: data-profiler"
  }
}
```

Allow only commands the plugin actually needs — plugin-internal scripts (`python ${CLAUDE_PLUGIN_ROOT}/skills/*/scripts/*.py`), specific CLI subcommands (`dbt run/test/build`, `fab auth/cd/ls/import`, `az account get-access-token`), filesystem discovery (`find -name "*.csv"`, `ls`), and a few utility patterns. Anything outside the allowlist falls through to the default permission flow — you are not broadly unlocking the shell.

#### Pattern B — PreToolUse Write/Edit structural validator (Source: Fabric N1)

**Problem:** Generated artifacts can violate the plugin's structural contract — wrong file extension, wrong target folder, etc. — and the orchestrator might cleanup-after-the-fact but the violations still ship.

**Fix:** Ship a hook that intercepts `Write` and `Edit` calls and blocks the ones that violate structure, with a clear remediation message:

```python
# hooks/validate-structure.py
import json, sys, re
event = json.load(sys.stdin)
path = event["tool_input"].get("file_path", "")

# Block .py writes to 3 - Notebooks/ — must be .ipynb (Fabric N1)
if re.search(r"3 - Notebooks/.*\.py$", path):
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": (
                "Fabric notebook deploy API treats .py as a single mega-cell. "
                "Use .ipynb (Jupyter JSON) instead."
            )
        }
    }))
    sys.exit(0)

# Default: allow
print(json.dumps({}))
```

The validator runs ahead of every Write/Edit and can deny + explain. This is belt-and-suspenders against agent-body drift (Source: Fabric N17 — bronze-builder's agent.md drifted to prescribe `.py` despite the hook).

#### Pattern C — SessionStart config-check

**Problem:** `userConfig` interactive prompt may silently skip on install (Source: DBT F5 Problem B), so users don't get asked for their values and the plugin starts broken.

**Fix:** Ship a SessionStart hook that detects missing userConfig env vars on every session start and prints a "Setup Required" message:

```json
"SessionStart": [
  {
    "matcher": "",
    "hooks": [
      { "type": "command",
        "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/session-start-config-check.py" }
    ]
  }
]
```

The hook should check for the bare env vars **and** the `CLAUDE_PLUGIN_OPTION_*` versions (since the user may have config but not have triggered the remap helper yet), and print exact instructions for hand-editing `~/.claude/settings.json` if both are missing.

### Worktree hooks: do not adopt blindly (Source: Fabric N8)

The DBT plugin originally shipped `WorktreeCreate` / `WorktreeRemove` hooks that isolated each background builder into its own git worktree. This works for dbt because dbt models are git-tracked text and the worktree commits them before removal. **It does not work for any agent that writes new untracked files** — the Fabric plugin's bronze builders write fresh `.ipynb` files, the `WorktreeRemove` hook ran `git worktree remove --force`, and every notebook the builder just produced was deleted milliseconds later. The builder's JSON envelope honestly reported success; the hook silently destroyed the output.

**Rule:** only adopt `isolation: worktree` and the matching hooks if both:

1. The agent writes files that are **already git-tracked**, AND
2. The `WorktreeRemove` hook explicitly **commits and merges back** to the main worktree *before* the `git worktree remove --force`.

Otherwise, skip worktree isolation and rely on unique per-agent filenames to avoid collisions. The Fabric plugin removed worktree hooks entirely and gave each builder a unique `nb_{layer}_{query}.ipynb` path.

---

## 9. Bundling an MCP Server into a Plugin (Optional)

If your plugin needs a long-running service that exposes tools (database introspection, API wrappers, custom protocols), bundle an MCP server. The DBT plugin's `sql-server-mcp` (Node, TypeScript) is the worked example.

### Ship compiled output

Do not expect users to run `npm install && npm build` after install. Ship a `servers/dist/` folder with the compiled JS bundle and point `plugin.json` at it:

```json
"mcpServers": {
  "sql-server-mcp": {
    "command": "node",
    "args": ["${CLAUDE_PLUGIN_ROOT}/servers/dist/minimal-mcp-server.js"],
    "env": {
      "SQL_SERVER": "${user_config.sql_server}",
      "SQL_DATABASE": "${user_config.sql_database}",
      "SQL_PASSWORD": "${user_config.sql_password}"
    }
  }
}
```

### Wire userConfig → server env explicitly

Every userConfig key the server reads needs an explicit mapping in `mcpServers.<name>.env`. This is the only path that gets the bare env var name (e.g. `SQL_SERVER`) into the server's `process.env`. Without it, the server starts with no config because the only env vars Claude Code exports are the prefixed `CLAUDE_PLUGIN_OPTION_<KEY>` form.

### Document Node version + prerequisites

In `README.md`, document the Node version requirement explicitly (`>= 18` for both shipped plugins) and any platform-specific quirks (e.g., ODBC driver requirements for SQL Server).

---

## 10. Reference Materials Inside a Plugin

Style guides, code examples, risk catalogs, conversion patterns — anything an agent needs to read at runtime — go in `reference/` at plugin root.

```
reference/
├── style-guide.md
├── testing-patterns.md
├── m-conversion-risk-catalog.md   # sentinel file for ancestor walk
└── examples/
    └── staging.sql
```

### Background subagents cannot read plugin-cache paths (Source: Fabric N14)

Background subagents run with restricted filesystem permissions. They **cannot read** files outside the project working directory — including paths inside `~/.claude/plugins/cache/<id>/reference/...`. If your orchestrator passes a `${CLAUDE_PLUGIN_ROOT}/reference/...` path to a background subagent, the subagent will hit a permissions block and the workflow stalls.

### Two patterns that work

**Pattern 1 — Copy at scaffold time (preferred for stable references):**

The plugin's project initializer copies `${CLAUDE_PLUGIN_ROOT}/reference/*` into a project-local folder (e.g. `6 - Agentic Resources/reference/`) at scaffold time. Subagent prompts thereafter reference the project-local copy:

```
# Wrong (background subagent can't read this):
${CLAUDE_PLUGIN_ROOT}/reference/risk-catalog.md

# Right (project-local, in subagent's working directory):
6 - Agentic Resources/reference/risk-catalog.md
```

Plus the stage ordering must put the project initializer **before** any subagent that reads the references.

**Pattern 2 — Embed inline in the prompt (preferred for small references):**

The orchestrator reads the reference file itself (its filesystem scope is unrestricted) and embeds the content directly in the spawned subagent's `prompt:` string. Good for small references; falls apart if the reference is more than a few thousand tokens.

### Validate the copy landed (Source: Fabric N16)

Use a sentinel file inside the reference folder (a specific filename the project depends on) and have the initializer verify the file is actually present after the copy. Don't trust `dir.exists()` — a half-populated stub folder will pass that check and ship broken.

---

## 11. The Marketplace (Distribution)

Both shipped plugins use a **two-repo distribution model**.

### Repo A — the plugin repo

Contains the plugin itself: `.claude-plugin/plugin.json`, `agents/`, `skills/`, `hooks/`, `servers/`, `reference/`. **No `marketplace.json`.** Example: [`DBT-Pipeline-Plugin`](https://github.com/KavasiMihaly/DBT-Pipeline-Plugin).

### Repo B — the marketplace repo

Contains only `.claude-plugin/marketplace.json` plus a `README.md`. **No plugin code.** Example: [`AI-plugins`](https://github.com/KavasiMihaly/AI-plugins).

```
AI-plugins/
├── .claude-plugin/
│   └── marketplace.json     # indexes plugins by URL
├── .gitignore
└── README.md
```

### marketplace.json shape

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "OneDayBI-Marketplace",
  "description": "AI-powered tools for data engineering and Power BI.",
  "owner": { "name": "Author", "email": "author@example.com" },
  "plugins": [
    {
      "name": "dbt-pipeline-toolkit",
      "description": "End-to-end dbt pipeline automation for SQL Server.",
      "source": {
        "source": "url",
        "url": "https://github.com/handle/DBT-Pipeline-Plugin.git"
      },
      "category": "development",
      "homepage": "https://github.com/handle/DBT-Pipeline-Plugin"
    }
  ]
}
```

The `source.source: "url"` field tells Claude Code to clone the plugin from a Git URL on install. The marketplace itself is just an index.

### Why two repos

- **Independent versioning** — plugin's `version` bumps don't pollute the marketplace history.
- **Independent issue tracking** — bug reports on `dbt-pipeline-toolkit` go to its own repo; meta-marketplace concerns (add my plugin to the index, broken URL, etc.) go to the marketplace repo.
- **One marketplace indexes many plugins** — keep adding new plugin repos and entries to the marketplace, users `/plugin marketplace add` once and see everything you ship.
- **Each plugin can be a marketplace too** — `DBT-Pipeline-Plugin` repo can be added directly as a marketplace (`/plugin marketplace add KavasiMihaly/DBT-Pipeline-Plugin`) for users who don't want the broader index.

### What the plugin repo's `README.md` says about the marketplace

A user-facing install flow needs at minimum:

```
# Add the marketplace
/plugin marketplace add KavasiMihaly/AI-plugins

# Install the plugin
/plugin install dbt-pipeline-toolkit@OneDayBI-Marketplace

# Reload
/reload-plugins
```

---

## 12. Installation Walkthrough

End-to-end, for a user installing your plugin:

```
# 1. Add the marketplace (once per machine)
/plugin marketplace add <author>/<marketplace-repo>

# 2. Install the plugin
/plugin install <plugin-name>@<marketplace-name>

# 3. Reload (or fully restart Claude Code if MCP servers don't appear)
/reload-plugins

# 4. Verify
/mcp        # should list any plugin-shipped MCP servers
/agents     # should list every plugin-shipped agent with full 3-part name
/skills     # should list every plugin-shipped skill with 2-part name
```

### Invocation patterns after install

For a main-thread orchestrator launch (the only path that supports delegation):

```
claude --agent <plugin>:<subdir>:<name> "<initial prompt>"
```

For a Skill invocation from any session:

```
/<plugin>:<skill> <args>
```

For a subagent spawn inside another agent's body:

```
Task(
  subagent_type: "<plugin>:<subdir>:<name>",
  prompt: "...",
  run_in_background: true,
  mode: "acceptEdits"
)
```

---

## 13. Testing — Fresh Install Is Non-Negotiable

The single biggest takeaway from shipping both plugins: **dev-mode testing hides the exact bugs your users will hit.**

### Why dev mode is a lie

Dev mode (`--plugin-dir ./`, or standalone agents in `~/.claude/agents/`, or skills in `~/.claude/skills/`) diverges from installed-mode along at least 8 axes — namespace, frontmatter stripping, hook/MCP support, script paths, env-var prefix, delegation rules, plugin-cache reads, and worktree hooks.

> **Canonical table:** See `claude-code-plugin-best-practices.md` § 1 ("The Core Insight: Dev vs Installed Behavior Diverges") for the full row-by-row breakdown with finding citations. That section is the authoritative reference for this divergence.

(Source: DBT F10)

### The bugs you'll only catch on fresh install

- 3-part namespacing on subdirectory agents (DBT F1)
- `permissionMode` stripping (DBT F2)
- Background subagents stalling on permission prompts (DBT F3)
- userConfig env var remap mismatch (DBT F5)
- `${CLAUDE_PLUGIN_ROOT}` substitution in markdown body (DBT F7)
- `skills:` frontmatter 2-part vs 3-part namespace mismatch (DBT F8)
- Worktree-remove hook destroying agent output (Fabric N8)
- Background subagent unable to read plugin-cache reference paths (Fabric N14)
- `AskUserQuestion` unavailable in subagents (Fabric N15)
- Reference-folder resolution stubbing on missing env var (Fabric N16)

Every one of those was invisible in dev. Every one fired on the first or second install attempt.

### How to test

Either:

1. **A clean second machine** that has never seen the repo. Install from the marketplace, run the happy path, see what breaks.
2. **Wipe `~/.claude/plugins/`** on your own machine, then install fresh from the marketplace URL.

Run the full happy path. **Verify side effects**, not just exit codes — orchestrators in background mode happily report `status: success` and produce nothing on disk (Fabric N8). After every stage, `ls` the expected output paths.

---

## 14. Pre-Shipment Audit

Build a `tests/preshipment_audit.py` and run it on every release. The Fabric plugin ships 7 gates as concrete examples:

| Gate | What it checks | Catches |
|---|---|---|
| `required_files` | `.claude-plugin/plugin.json`, `README.md`, every agent has `agent.md`, every skill has `SKILL.md` | Missing essential files |
| `plugin_manifest` | `plugin.json` parses as JSON, has required fields, no schema violations | Manifest typos that would break load |
| `paths` | No `$HOME/.claude/skills/` references in any `*.md` under `agents/` or `skills/`; all path refs use `${CLAUDE_PLUGIN_ROOT}` or `${CLAUDE_SKILL_DIR}` | DBT F7 regressions |
| `namespace` | Every `subagent_type:` and `Agent(...)` allowlist entry uses 3-part name; every `skills:` field uses 2-part name | DBT F1 and F8 regressions |
| `skills_frontmatter` | Every agent's `skills:` field uses 2-part `<plugin>:<skill>` form | DBT F8 regressions |
| `atomic_bash` | No `&&`, `\|\|`, `;`, `\|`, subshells, or heredocs in any Bash code block across all `agents/*/agent.md` and `skills/*/SKILL.md` | DBT F9 compound-command regressions |
| `notebook_extension` | (Fabric-specific) No agent body line prescribes a `.py` notebook output under `3 - Notebooks/` | Fabric N17 drift |

Each gate is a function that walks the relevant files and prints `[PASS]` / `[FAIL]` with the offending file/line. The audit script exits non-zero on any FAIL.

```
=== Pre-shipment audit: fabric-dataflow-migration-toolkit ===
  [PASS] required_files
  [PASS] plugin_manifest
  [PASS] paths
  [PASS] namespace
  [PASS] skills_frontmatter
  [PASS] atomic_bash
  [PASS] notebook_extension
Overall: PASS
```

### Wire it into CI

Block PRs that fail the audit. New gates are cheap to add — each one is a function that catches a single class of regression. Every finding in `_Documentation/plugin_learnings.md` that has a mechanical signature deserves its own gate. Aim for "the audit catches it before it ships," not "the user catches it after."

---

## 15. Versioning & Release Process

### Release steps

1. Update `_Documentation/plugin_learnings.md` with any new findings from the current cycle.
2. Update `_Plan/Issues.md` — close issues that this release fixes, add follow-ups for empirical verifications.
3. Bump `version` in `plugin.json` (semver: `major.minor.patch`).
4. Run `tests/preshipment_audit.py` — must PASS all gates.
5. Test on a fresh install (Section 13).
6. Tag the plugin repo: `git tag v1.2.0 && git push --tags`.
7. Push to main.

### What users do

On the next `/plugin` or `/reload-plugins`, the marketplace re-fetches and detects the new version. Users may need to fully restart Claude Code for MCP server changes (the running MCP process won't reload mid-session).

### Breaking-change policy

Document this in `CLAUDE.md` so contributors don't ship silent breakage:

- **Manifest-breaking changes** (renaming an agent, changing a userConfig key, removing an MCP server tool) bump the **major** version.
- **Agent / skill behavior changes that affect external invocations** (different argument order on a CLI flag, different output format) bump the **minor** version with a changelog note.
- **Internal fixes that don't change the external contract** bump the **patch** version.

Users with `/plugin install <name>@<marketplace>` automatically get the latest version on next reload. There's no built-in version pinning — if you ship a breaking change, document the migration path prominently in `README.md` and pin the change clearly in the changelog.

---

## 16. Common Pitfalls

A quick-triage table for someone hitting a weird bug. Each row points back to the canonical finding in the source plugin's `plugin_learnings.md`.

| Pitfall | Mitigation | Source |
|---|---|---|
| Subdirectory agents register with 3-part name, not 2 — bare-name references silently no-op | Use full `<plugin>:<subdir>:<name>` everywhere; subdir name must equal frontmatter `name` field | DBT F1 |
| `permissionMode`, `hooks`, `mcpServers` in agent frontmatter silently stripped | Pass `mode:` at call site; declare hooks/MCP at plugin level in `plugin.json` | DBT F2 |
| Background subagents stall indefinitely on permission prompts | Always pass `mode: "acceptEdits"` in `Task(..., run_in_background: true)` | DBT F3 |
| Orchestrator launched via slash command becomes a leaf subagent and can't delegate | Only launch via `claude --agent <plugin>:<subdir>:<name>`; document this in README | DBT F4, Fabric N11 |
| `userConfig` env vars exported as `CLAUDE_PLUGIN_OPTION_<KEY>`, not bare `<KEY>` | Add `_load_plugin_userconfig_env()` helper to every script before argparse | DBT F5 (Problem A) |
| `userConfig` interactive prompt may silently skip on install | Ship SessionStart hook + document manual `settings.json` paths in README | DBT F5 (Problem B) |
| `$HOME/.claude/skills/...` paths in agent / SKILL.md don't exist on fresh install | Use `${CLAUDE_PLUGIN_ROOT}/skills/<name>/scripts/<file>.py` everywhere | DBT F7 |
| `skills:` frontmatter with bare skill names resolves to nothing in plugin-shipped agents | Use 2-part `<plugin>:<skill>` form for `skills:` field (different from agent's 3-part) | DBT F8 |
| Background subagents can't run arbitrary Bash (Python scripts, CLIs) — stalls silently | Ship plugin-level PreToolUse Bash allowlist hook | DBT F9 |
| Compound Bash (`A && B`, `cmd \| jq`) breaks permission matching and hook splitter | Refactor every command to atomic single-operation form | DBT F9 (Round 2) |
| Dev-mode behavior diverges from install-mode | Test only on a fresh install or wiped `~/.claude/plugins/` | DBT F10 |
| Worktree-remove hook deletes builder output when files aren't git-tracked | Drop worktree isolation OR add commit-and-merge-back to WorktreeRemove hook | Fabric N8 |
| Interactive subagent spawned in background — `AskUserQuestion` silently no-ops | Always pass explicit `run_in_background: false` for interactive subagents | Fabric N9 |
| `EnterPlanMode` not in orchestrator `tools:` — plan-mode silently no-ops | Use `AskUserQuestion` with Approve / Revise / Abort options instead | Fabric N10 |
| Slash command spawns orchestrator → orchestrator's `Task()` calls no-op | Delete slash command; document `claude --agent ...` as canonical entry | Fabric N11 |
| Corporate TLS interception breaks every Python-based CLI (`az`, `fab`, `pip`) | Document `REQUESTS_CA_BUNDLE` setup; prefer Windows-native auth paths | Fabric N12 |
| Generated PowerShell `.ps1` files fail under PS 5.1 due to UTF-8 without BOM | Write all generated PowerShell as UTF-8 with BOM (`utf-8-sig`); avoid non-ASCII in string literals | Fabric N13 |
| Background subagents can't read `${CLAUDE_PLUGIN_ROOT}/reference/...` paths | Copy references into project at scaffold time; or embed inline in prompt | Fabric N14 |
| `AskUserQuestion` unavailable in any subagent regardless of foreground/background | Parent owns interactive Q&A; subagents emit JSON envelopes for the parent to consume | Fabric N15 |
| Env-var-only resource resolution falls back to non-existent path when var unset | Use sentinel-file ancestor walk; degrade loudly, never with stub content | Fabric N16 |
| Agent body content drifts from hook/structure rules over time | Add pre-shipment audit gate that mirrors every runtime hook check | Fabric N17 |

---

## 17. Companion Documents

| Document | What it covers |
|---|---|
| [`claude-code-plugin-best-practices.md`](claude-code-plugin-best-practices.md) | Patterns and design rules for plugin architecture (companion to this how-to) |
| [`claude-code-agent-best-practices.md`](claude-code-agent-best-practices.md) | Agent-level patterns including a plugin section |
| [`claude-code-skill-best-practices.md`](claude-code-skill-best-practices.md) | Skill-level patterns including a plugin section |
| [`claude-code-hooks-best-practices.md`](claude-code-hooks-best-practices.md) | Hook patterns including plugin-level hook design |
| [`how-to-create-agents.md`](how-to-create-agents.md) | Step-by-step guide for standalone agents |
| [`how-to-create-skills.md`](how-to-create-skills.md) | Step-by-step guide for standalone skills |

Source plugin learnings (the primary evidence base for this guide) live in the author's plugin repos:

- [`KavasiMihaly/DBT-Pipeline-Plugin`](https://github.com/KavasiMihaly/DBT-Pipeline-Plugin) — see `_Documentation/plugin_learnings.md`, findings F1 through F10
- [`KavasiMihaly/Dataflow-to-Notebook-Plugin`](https://github.com/KavasiMihaly/Dataflow-to-Notebook-Plugin) — see `_Documentation/plugin_learnings.md`, findings N1 through N17 plus inherited F1–F9

When a finding in this guide cites `DBT Fn` or `Fabric Nn`, that's the file and section to read for the full narrative — every claim here is backed by an actually-shipped, actually-broken-then-fixed bug in one of those two plugins.

### External research (cross-validation against official sources)

The plugin best-practices doc (`claude-code-plugin-best-practices.md`) has a Source Validation section that cross-references every finding in this how-to against:

All external citations (changelog entries, Anthropic blog quotes, expert YouTube references) are consolidated into § 15–18 of `claude-code-plugin-best-practices.md`.

Findings without official corroboration (still flagged as "no public release note found" in the Source Validation table): F1 (exact 3-part namespace format), N1 (Fabric `.ipynb`-only rule), N11 (slash-commands-can't-host-orchestrators), N12/N13 (corporate TLS / PowerShell encoding — Microsoft / industry behavior, not Anthropic).
