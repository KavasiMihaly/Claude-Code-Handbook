# Claude Code Commands & Settings — Quick Reference

**Last Updated**: May 2026
**Purpose**: The slash commands worth knowing and the `settings.json` setup that has no UI — including MCP wiring and the keys you can only reach by editing the file.

---

## Table of Contents

1. [Slash Commands That Earn Their Keep](#slash-commands-that-earn-their-keep)
2. [The settings.json Hierarchy](#the-settingsjson-hierarchy)
3. [Adding MCP Servers](#adding-mcp-servers)
4. [Plugins — Config & Control](#plugins--config--control)
5. [Settings Keys With No UI](#settings-keys-with-no-ui)
6. [Useful Environment Variables](#useful-environment-variables)
7. [Cheat Sheet](#cheat-sheet)

---

## Slash Commands That Earn Their Keep

Type `/` in any session to see the full list. These are the ones worth committing to muscle memory, grouped by what you're trying to do.

### Context & token management

| Command | What it does | Why it matters |
|---|---|---|
| `/context` | Renders current context-window usage as a colored grid, with what's eating it (system prompt, MCP tools, files, history). | The single best habit. Run it when responses feel sluggish — you'll often find an MCP server or a giant file is the culprit. |
| `/compact [instructions]` | Summarizes the conversation so far to reclaim context, then continues the same task. Optional instructions steer what to keep. | Use *before* you hit the wall, not after. `/compact keep the repro steps and failing test` preserves what matters. |
| `/clear` | Starts a fresh conversation. Project memory (CLAUDE.md) stays; history is recoverable via `/resume`. | Cheapest way to fix a derailed session. New task = `/clear`. |

### Configuration & control

| Command | What it does |
|---|---|
| `/agents` | Create, edit, and inspect subagents (custom agent definitions, model, tools, isolation). The UI front-end for `agent.md` files. |
| `/mcp` | List MCP servers, see connection status, and run OAuth sign-in flows for servers that need it. |
| `/permissions` | View and edit allow / ask / deny rules; review what auto-mode recently blocked. |
| `/hooks` | Inspect which hooks are registered for each lifecycle event. |
| `/config` | Interactive settings UI — theme, model, output style, editor mode, and other surface preferences. |
| `/model <name>` | Switch model (and set it as the session default). |
| `/memory` | Open and edit the CLAUDE.md memory files that load each session. |

### Session lifecycle

| Command | What it does |
|---|---|
| `/init` | Generate a starter `CLAUDE.md` by scanning the repo. First thing to run in a new project. |
| `/resume` | Reopen a previous conversation by picker or ID. |
| `/rewind` | Roll the conversation **and** file changes back to an earlier checkpoint. Your undo button for an agent that went sideways. |
| `/export` | Dump the conversation to a text file. |
| `/cost` (`/usage`, `/status`) | Show spend, plan limits, and an activity breakdown. |

### Code review & quality

| Command | What it does |
|---|---|
| `/review [PR]` | Review a pull request (or the local diff) for bugs and cleanups. |
| `/security-review` | Scan pending changes for vulnerabilities — injection, auth gaps, data exposure. |

### Diagnostics

| Command | What it does |
|---|---|
| `/doctor` | Verify the install and config; can auto-fix common problems. |
| `/help` | List available commands. |

> **Highest-value five:** `/context`, `/compact`, `/clear`, `/agents`, `/permissions`. Master these and the rest are discoverable.

> Command availability tracks your Claude Code version — newer builds add commands (background agents, batch, scheduling, remote control). Run `/help` to see exactly what your install supports.

---

## The settings.json Hierarchy

Settings cascade across scopes. When the same key appears in more than one place, the **higher tier wins**:

| Precedence | Scope | Path | Shared? |
|---|---|---|---|
| 1 (highest) | **Managed / enterprise** | OS-specific managed-policy dir (e.g. `/etc/claude-code/managed-settings.json` on Linux, `C:\Program Files\ClaudeCode\` on Windows) | IT-deployed, non-overridable |
| 2 | **Command-line flags** | `claude --model … --permission-mode …` | Session only |
| 3 | **Project local** | `<repo>/.claude/settings.local.json` | No — auto-gitignored |
| 4 | **Project shared** | `<repo>/.claude/settings.json` | Yes — committed for the team |
| 5 (lowest) | **User** | `~/.claude/settings.json` | No — personal default |

Rule of thumb: **personal defaults** → user settings; **team conventions** → project settings (committed); **machine-specific secrets/overrides** → local settings (never committed).

A minimal `~/.claude/settings.json`:

```json
{
  "model": "claude-opus-4-6",
  "includeCoAuthoredBy": true,
  "permissions": {
    "allow": ["Read", "Grep", "Glob"],
    "ask": ["Bash"],
    "deny": ["Read(./.env)", "Read(./secrets/**)"]
  }
}
```

---

## Adding MCP Servers

MCP servers are configured as JSON, not through a UI. Three ways to register one:

### 1. Project servers — `.mcp.json` (committed, shared with the team)

Lives at the repo root. Everyone who trusts the project gets the same servers.

```json
{
  "mcpServers": {
    "local-sql": {
      "type": "stdio",
      "command": "node",
      "args": ["./servers/sql/index.js"],
      "env": { "SQL_CONN": "Server=localhost;Trusted_Connection=yes" }
    },
    "docs-http": {
      "type": "http",
      "url": "https://example.com/mcp"
    },
    "events-sse": {
      "type": "sse",
      "url": "https://example.com/sse"
    }
  }
}
```

### 2. The `claude mcp add` CLI

Fastest for a quick add without hand-editing JSON:

```bash
claude mcp add my-server --scope project -- node ./servers/my/index.js
```

`--scope` is `user`, `project`, or `local`. Run `claude mcp list` to see what's registered and `/mcp` inside a session to check connection/auth status.

### Transport types

| `type` | Shape | Use when |
|---|---|---|
| `stdio` | `command` + `args` + optional `env` | Server is a local process (the common case). |
| `http` | `url` | Server is a remote HTTP endpoint. |
| `sse` | `url` | Server streams over Server-Sent Events. |

### Controlling which `.mcp.json` servers load

Because project MCP servers run code, Claude Code asks before trusting them. Pre-approve or gate them from `settings.json`:

| Key | Type | Effect |
|---|---|---|
| `enableAllProjectMcpServers` | boolean | Auto-approve every server in `.mcp.json`. |
| `enabledMcpjsonServers` | string[] | Allowlist specific servers by name. |
| `disabledMcpjsonServers` | string[] | Blocklist specific servers by name. |

---

## Plugins — Config & Control

A **plugin** is a self-contained bundle of skills, agents, hooks, and MCP servers (`.claude-plugin/plugin.json`). A **marketplace** is a catalog that lists many plugins (`.claude-plugin/marketplace.json`). You add a marketplace once, then install plugins from it. Both are recorded in `settings.json`, so you can manage them by hand as well as through the UI.

### The `/plugin` command

`/plugin` opens the interactive manager with tabs for **Discover** (browse), **Installed** (enable / disable / uninstall, with a scope selector), **Marketplaces** (add / remove / update), and **Errors** (load failures). Marketplace management has its own subcommands:

```text
/plugin marketplace add <owner/repo | git-url | ./local-path | https://…/marketplace.json>
/plugin marketplace list
/plugin marketplace update <name>
/plugin marketplace remove <name>
```

There's a matching headless CLI for scripts and CI (each takes `--scope user|project|local`):

```bash
claude plugin install   <plugin>@<marketplace> --scope project
claude plugin enable     <plugin>@<marketplace>
claude plugin disable    <plugin>@<marketplace>
claude plugin uninstall  <plugin>@<marketplace> [--keep-data] [--prune]
claude plugin list [--json] [--available]
claude plugin marketplace add <source>
```

### How plugins are recorded in settings.json

Three keys, all editable by hand. They live at whatever scope the plugin was installed into (user `~/.claude/settings.json`, project `.claude/settings.json`, or local `.claude/settings.local.json`):

| Key | Shape | What it does |
|---|---|---|
| `extraKnownMarketplaces` | object keyed by marketplace name | Registers a marketplace so its plugins are installable. |
| `enabledPlugins` | object keyed by `"plugin@marketplace"` | The installed-and-enabled set. Value is `{}` (reserved). |
| `pluginConfigs` | object keyed by `"plugin@marketplace"` | Per-plugin settings under an `options` block (the values the install prompt collects). |

```json
{
  "extraKnownMarketplaces": {
    "onedaybi": {
      "source": { "source": "github", "repo": "KavasiMihaly/AI-plugins" },
      "autoUpdate": true
    }
  },
  "enabledPlugins": {
    "dbt-pipeline-toolkit@onedaybi": {}
  },
  "pluginConfigs": {
    "dbt-pipeline-toolkit@onedaybi": {
      "options": {
        "sql_server": "localhost",
        "sql_database": "Agentic"
      }
    }
  }
}
```

`source` also accepts a bare URL string (`"source": "https://gitlab.com/org/plugins.git"`) instead of the `{ source, repo }` object.

> **The manual-config fallback.** Plugin install prompts sometimes don't fire on an install/update. When that happens, the plugin can't find its settings — hand-edit `pluginConfigs.<plugin@marketplace>.options.<key>` to fix it. Values marked `"sensitive": true` in the plugin's `userConfig` are **not** written here; they go to the OS keychain. Each option is also exported to plugin subprocesses as `CLAUDE_PLUGIN_OPTION_<KEY>` and is substitutable as `${user_config.<key>}` inside the plugin's MCP/hook configs.

### Enterprise / managed control (admin-only)

Set in the managed-policy file (non-overridable by users) to govern what plugins a fleet may run:

| Key | Type | Effect |
|---|---|---|
| `strictKnownMarketplaces` | array | Allowlist of permitted marketplace sources. Empty array = total lockdown; undefined = no restriction. |
| `blockedMarketplaces` | array | Denylist of marketplace sources — takes precedence over the allowlist. |
| `enabledPlugins` (managed form) | array of `{ marketplace, plugin }` | Force-enable plugins for everyone; their hooks load even under `allowManagedHooksOnly`. |
| `disabledPlugins` | array of `{ marketplace, plugin }` | Force-disable plugins; users can't turn them back on. |
| `strictPluginOnlyCustomization` | boolean \| string[] | Block skills/agents/hooks/MCP from user & project sources — only plugins or managed settings may provide them. |
| `allowManagedHooksOnly` | boolean | Only managed (and force-enabled-plugin) hooks run; user/project hooks are ignored. |

> Note: `enabledPlugins` is an **object** (`{"plugin@marketplace": {}}`) in user/project scope but an **array of `{marketplace, plugin}`** in managed scope. The managed plugin-governance keys are recent additions — confirm against your CLI version with `/doctor`.

---

## Settings Keys With No UI

These have no `/config` entry — you set them by editing `settings.json` directly. This is where most of the real power lives.

### Permissions

| Key | Type | Description |
|---|---|---|
| `permissions.allow` | string[] | Tools that run without prompting. Format `"Tool"` or `"Tool(specifier)"`, e.g. `"Bash(npm run test:*)"`. |
| `permissions.ask` | string[] | Tools that always prompt (evaluated after deny/allow). |
| `permissions.deny` | string[] | Tools that are blocked outright — first match wins. Great for `Read(./.env)`. |
| `permissions.defaultMode` | string | Starting permission mode: `default`, `acceptEdits`, `plan`, `bypassPermissions`. |
| `permissions.additionalDirectories` | string[] | Extra directories Claude may touch beyond the project root. |

### Automation & display

| Key | Type | Description |
|---|---|---|
| `hooks` | object | Map lifecycle events (`PreToolUse`, `PostToolUse`, `Stop`, …) to commands. The only way to make Claude Code run *your* code automatically. |
| `statusLine` | object | Custom status line from a script: `{ "type": "command", "command": "~/.claude/statusline.py" }`. |
| `outputStyle` | string | Persisted response style (also reachable in `/config`, but settable here). |
| `env` | object | Environment variables injected into every session. |

### Models & auth

| Key | Type | Description |
|---|---|---|
| `model` | string | Default model. |
| `apiKeyHelper` | string | Path to a script that prints a short-lived API key (rotating creds). |
| `awsAuthRefresh` / `awsCredentialExport` | string | Scripts to refresh AWS creds for Bedrock. |
| `forceLoginMethod` | string | Restrict sign-in to `claudeai` or `console`. |

### Housekeeping & git

| Key | Type | Description |
|---|---|---|
| `includeCoAuthoredBy` | boolean | Add the `Co-Authored-By: Claude` trailer to commits (default true). |
| `cleanupPeriodDays` | number | How long to retain transcripts / debug state before auto-deletion. |

> Some keys above are version-dependent and a few advanced ones (sandbox, worktree, marketplace controls) appear only in recent builds. Check `/doctor` and your version's docs if a key seems to do nothing.

---

## Useful Environment Variables

Some behavior is only reachable through environment variables (no `settings.json` key). Set them in your shell profile or in the `env` block of `settings.json`.

| Variable | Description |
|---|---|
| `ANTHROPIC_API_KEY` | API key (for direct API billing). |
| `ANTHROPIC_MODEL` | Override the default model globally. |
| `CLAUDE_CODE_USE_BEDROCK` | Route through Amazon Bedrock. |
| `MAX_THINKING_TOKENS` | Fix the extended-thinking token budget. |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Cap output tokens per response. |
| `BASH_DEFAULT_TIMEOUT_MS` | Default Bash command timeout (default 120000). |
| `API_TIMEOUT_MS` | API request timeout (default 600000). |
| `DISABLE_TELEMETRY` / `DO_NOT_TRACK` | Opt out of telemetry. |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Stop loading CLAUDE.md files (debugging). |

---

## Cheat Sheet

```text
Feels slow?           /context  →  /compact  →  (new task) /clear
New repo?             /init
Build an agent?       /agents
Wire up a tool?       edit .mcp.json  (or claude mcp add)  →  /mcp to verify
Add a plugin?         /plugin marketplace add <owner/repo>  →  /plugin  →  install
Plugin prompt skip?   edit pluginConfigs.<plugin@market>.options in settings.json
Lock down a tool?     permissions.deny in settings.json
Automate on events?   hooks in settings.json
Went sideways?        /rewind
Something broken?     /doctor
```

---

*Part of the [Claude Code Handbook](README.md). Slash-command names and settings keys track the CLI version — run `/help` and `/doctor` to confirm what your build supports.*
