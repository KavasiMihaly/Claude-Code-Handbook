# How to Create Claude Code Hooks

**Last Updated:** 2026-05-25
**Purpose:** Step-by-step guide for building, registering, and testing Claude Code hooks across all 29 events.

This guide is one of four how-tos alongside `how-to-create-agents.md`, `how-to-create-skills.md`, and `how-to-create-plugins.md`. It covers hook events, the JSON protocol, configuration scopes (user / project / plugin), and worked examples for the most common hook patterns. For *why* and *when* to use each pattern, plus shipped-plugin lessons, see the companion `claude-code-hooks-best-practices.md`.

## Table of Contents

1. [What Are Hooks?](#1-what-are-hooks)
2. [The Hook Event Catalogue (29 events)](#2-the-hook-event-catalogue-29-events)
3. [Hook Script Anatomy (JSON contract)](#3-hook-script-anatomy-json-contract)
4. [Configuration Scopes](#4-configuration-scopes)
5. [Step-by-Step Creation Process](#5-step-by-step-creation-process)
6. [Common Hook Patterns](#6-common-hook-patterns)
7. [Testing Hooks](#7-testing-hooks)
8. [Cross-Platform Considerations (Windows + Linux/macOS)](#8-cross-platform-considerations-windows--linuxmacos)
9. [Plugin-Level Hooks (registration in plugin.json)](#9-plugin-level-hooks-registration-in-pluginjson)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. What Are Hooks?

**Hooks** are user-defined commands (or LLM prompts) that fire at specific points in the Claude Code lifecycle. Unlike `CLAUDE.md` rules, which the model can ignore or "forget" as context fills, hooks are deterministic — they are executed by the harness, not by Claude. Anthropic frames the distinction explicitly:

> *"Hooks run scripts automatically at specific points in Claude's workflow. Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens."* — `code.claude.com/docs/en/best-practices`

Hooks were introduced in Claude Code v1.0.59 (June 30, 2025) and were folded into the autonomy bundle (hooks + subagents + background tasks + checkpoints) on 2025-09-29. They are one of the four payload types a plugin can ship — "Plugins are a lightweight way to package and share any combination of: Slash commands, Subagents, MCP servers, and Hooks."

### Five hook handler types

| Type | What it does | Typical use |
|------|-------------|-------------|
| `command` | Runs a shell command | 90% of hooks — validation, logging, formatting |
| `prompt` | Runs a single-turn LLM evaluation (Haiku by default) | Lightweight classifier-style decisions |
| `agent` | Spawns a multi-turn subagent with tool access | Heavy verification (run tests, audit code) |
| `http` | POSTs JSON to a URL | Streaming hook events to a dashboard |
| `mcp_tool` | Calls an MCP tool directly | Plugin-shipped MCP integrations (v2.1.118+) |

### Key benefits

1. **Determinism** — the action always happens; you don't have to hope Claude obeys a rule.
2. **Defense in depth** — `PreToolUse` is the only place to block a tool call regardless of what an agent or subagent decides.
3. **Observability** — hook output streams give you a JSONL audit trail of every tool call.
4. **Shareable enforcement** — bundle hooks into plugins and an entire team gets the same guardrails.

### When to use hooks vs. CLAUDE.md vs. permission rules

| Need | Use |
|------|-----|
| "Claude should usually do X" | CLAUDE.md rule |
| "Claude must always do X" | Hook (PostToolUse / Stop) |
| "Claude may not do Y" | Hook (PreToolUse with `permissionDecision: "deny"`) or `permissions.deny` rule |
| "Approve Y without prompting for known-safe cases" | Hook (PreToolUse with `permissionDecision: "allow"`) |

---

## 2. The Hook Event Catalogue (29 events)

The current set of 29 plugin-reachable hook events as of v2.1.150, sourced directly from `code.claude.com/docs/en/plugins-reference`. Use this as the lookup when choosing where to bind logic.

### Session lifecycle

| Event | When it fires | Matcher | Can block? |
|-------|--------------|---------|------------|
| `SessionStart` | Session begins or resumes | `startup`, `resume`, `clear`, `compact` | No |
| `Setup` | One-time prep with `--init-only` / `--init` / `--maintenance` in `-p` mode (CI / scripts) | — | No |
| `SessionEnd` | Session terminates | `clear`, `logout`, `prompt_input_exit`, `bypass_permissions_disabled`, `other` | No |
| `Stop` | Claude finishes responding | — | Yes (`decision: "block"`) |
| `StopFailure` | Turn ends due to API error | — | No |

### Prompt and instruction loading

| Event | When it fires | Matcher | Can block? |
|-------|--------------|---------|------------|
| `UserPromptSubmit` | User submits a prompt, before Claude processes | — | Yes (`decision: "block"`) |
| `UserPromptExpansion` | A user-typed command expands into a prompt | — | Yes |
| `InstructionsLoaded` | A `CLAUDE.md` or `.claude/rules/*.md` file loads into context. Fires at session start *and* on lazy loads during a session | — | No |

### Tool use lifecycle

| Event | When it fires | Matcher | Can block? |
|-------|--------------|---------|------------|
| `PreToolUse` | Before a tool call executes | Tool-name regex (`Bash`, `Edit\|Write`, `mcp__.*`) | Yes (`permissionDecision: "deny"`) |
| `PermissionRequest` | Permission dialog appears | Tool name | Yes (`behavior: "deny"`) |
| `PermissionDenied` | Tool call denied by auto-mode classifier. Return `{retry: true}` to tell the model to retry | Tool name | — |
| `PostToolUse` | After a tool succeeds | Tool name | No (already executed) |
| `PostToolUseFailure` | After a tool fails | Tool name | No |
| `PostToolBatch` | After a full batch of parallel tool calls resolves, before the next model call | — | No |
| `Notification` | Claude sends a notification | `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog` | No |

### Subagent / task lifecycle

| Event | When it fires | Matcher | Can block? |
|-------|--------------|---------|------------|
| `SubagentStart` | A subagent is spawned | Agent type (`Bash`, `Explore`, `Plan`, custom) | No |
| `SubagentStop` | A subagent finishes | Agent type | Yes (`decision: "block"`) |
| `TaskCreated` | A task is being created via `TaskCreate` | — | No |
| `TaskCompleted` | A task is being marked completed | — | Yes (exit 2) |
| `TeammateIdle` | An agent-team teammate is about to go idle | — | Yes (exit 2) |

### Environment / config

| Event | When it fires | Matcher | Can block? |
|-------|--------------|---------|------------|
| `ConfigChange` | A configuration file changes during a session | — | No |
| `CwdChanged` | The working directory changes | — | No |
| `FileChanged` | A watched file changes on disk | — | No |
| `WorktreeCreate` | A worktree is being created via `--worktree` or `isolation: 'worktree'`. Replaces default git behavior | — | Yes |
| `WorktreeRemove` | A worktree is being removed | — | Yes |

### Compaction

| Event | When it fires | Matcher | Can block? |
|-------|--------------|---------|------------|
| `PreCompact` | Before context compaction | `manual`, `auto` | No |
| `PostCompact` | After context compaction completes | — | No |

### MCP elicitation

| Event | When it fires | Matcher | Can block? |
|-------|--------------|---------|------------|
| `Elicitation` | An MCP server requests user input during a tool call | — | No |
| `ElicitationResult` | After a user responds to an MCP elicitation | — | No |

The original 14-event catalogue documented in mid-2025 community sources (`SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `PostToolUse`, `PostToolUseFailure`, `Notification`, `SubagentStart`, `SubagentStop`, `Stop`, `TeammateIdle`, `TaskCompleted`, `PreCompact`, `SessionEnd`) is now a subset of the above. The 15 additional events (`Setup`, `UserPromptExpansion`, `InstructionsLoaded`, `PermissionDenied`, `PostToolBatch`, `TaskCreated`, `StopFailure`, `ConfigChange`, `CwdChanged`, `FileChanged`, `WorktreeCreate`, `WorktreeRemove`, `PostCompact`, `Elicitation`, `ElicitationResult`) shipped quietly across v2.1.111 through v2.1.150 — most never received their own release-note bullet.

---

## 3. Hook Script Anatomy (JSON contract)

Every hook follows the same I/O contract: read JSON from stdin, emit JSON on stdout, choose an exit code.

### Input (stdin JSON)

Every hook receives a JSON payload with common fields plus event-specific fields.

**Common fields (all events):**

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default|plan|acceptEdits|dontAsk|bypassPermissions",
  "hook_event_name": "EventName"
}
```

**PreToolUse — Bash example:**

```json
{
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test",
    "description": "Run test suite",
    "timeout": 120000,
    "run_in_background": false
  },
  "tool_use_id": "toolu_01ABC123..."
}
```

**PreToolUse — Task/subagent example:**

```json
{
  "tool_name": "Task",
  "tool_input": {
    "prompt": "Research hooks documentation",
    "description": "Research hooks docs",
    "subagent_type": "general-purpose",
    "model": "sonnet"
  }
}
```

**UserPromptSubmit:**

```json
{ "prompt": "Write a function to calculate factorial" }
```

**SessionStart:**

```json
{
  "source": "startup|resume|clear|compact",
  "model": "claude-sonnet-4-6",
  "agent_type": "AgentName"
}
```

**SubagentStop:**

```json
{
  "agent_id": "abc123",
  "agent_type": "Explore",
  "agent_transcript_path": "/path/to/subagent-transcript.jsonl",
  "stop_hook_active": false,
  "background_tasks": [],
  "session_crons": []
}
```

`background_tasks` and `session_crons` were added to Stop/SubagentStop in v2.1.145.

**Tool input schemas by tool name:**

| Tool | Key input fields |
|------|-----------------|
| `Bash` | `command`, `description`, `timeout`, `run_in_background` |
| `Write` | `file_path`, `content` |
| `Edit` | `file_path`, `old_string`, `new_string`, `replace_all` |
| `Read` | `file_path`, `offset`, `limit` |
| `Glob` | `pattern`, `path` |
| `Grep` | `pattern`, `path`, `glob`, `output_mode`, `-i`, `multiline` |
| `WebFetch` | `url`, `prompt` |
| `WebSearch` | `query`, `allowed_domains`, `blocked_domains` |
| `Task` | `prompt`, `description`, `subagent_type`, `model` |

### Output (stdout JSON + exit codes)

**Exit codes:**

| Code | Meaning |
|------|---------|
| `0` | Success. JSON on stdout is processed normally. |
| `2` | Blocking error. stderr text is fed back to Claude as feedback. |
| other | Non-blocking error. stderr shown only in verbose mode (`Ctrl+O`). |

**Universal output fields** (any event):

```json
{
  "continue": true,
  "stopReason": "Message shown to user",
  "suppressOutput": false,
  "systemMessage": "Warning text injected into context"
}
```

**PreToolUse output (`hookSpecificOutput`):**

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "Plugin-internal Python script",
    "updatedInput": { "field": "new_value" },
    "additionalContext": "Context injected for Claude"
  }
}
```

**PermissionRequest output:**

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow|deny",
      "updatedInput": { "field": "value" },
      "message": "Reason for deny"
    }
  }
}
```

**Stop / SubagentStop output (block stopping):**

```json
{
  "decision": "block",
  "reason": "Tasks are not complete yet"
}
```

**PostToolUse output (replace tool output, v2.1.121+):**

```json
{
  "updatedToolOutput": "Filtered or augmented content"
}
```

**terminalSequence field (v2.1.141+):**

```json
{ "terminalSequence": "]0;Build passing" }
```

Lets hooks emit ANSI / OSC sequences for window titles, desktop notifications, and bells.

### Exec-form args (v2.1.139+)

Hooks can use exec-form args instead of a shell-string concatenation, eliminating shell-escaping pain:

```json
{
  "type": "command",
  "command": "python",
  "args": ["${CLAUDE_PLUGIN_ROOT}/hooks/validate.py", "--strict"]
}
```

---

## 4. Configuration Scopes

Hooks are registered in JSON settings files. Load order goes from least to most specific — later scopes override earlier ones.

| Scope | File | Shareable | Use case |
|-------|------|-----------|----------|
| **Managed policy** | OS-specific managed policy file | Yes (org-wide) | Admin-enforced hooks; admin can set `allowManagedHooksOnly: true` to disable all user/project/plugin hooks |
| **User-global** | `~/.claude/settings.json` | No (personal) | Personal habits — notifications, logging, safety |
| **Project-shared** | `.claude/settings.json` | Yes (commit) | Team standards — formatting, protected files, CI checks |
| **Project-local** | `.claude/settings.local.json` | No (gitignored) | Personal project overrides |
| **Plugin** | `plugin.json` `hooks` block | Yes (installed plugin) | Plugin-bundled hooks; visible in `/plugin` info |
| **Agent / Skill frontmatter** | YAML frontmatter — *only for user-defined agents/skills* | Yes | Component-scoped hooks while the component is active |

**Plugin-shipped agents cannot declare `hooks:` in their frontmatter** — for security, that field is silently stripped at install time. Plugin hooks must live in `plugin.json`. (See section 9.)

### Standard JSON structure

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "regex_pattern",
        "hooks": [
          {
            "type": "command",
            "command": "shell command here",
            "timeout": 600,
            "statusMessage": "Custom spinner message",
            "async": false
          }
        ]
      }
    ]
  }
}
```

**Field reference:**

- `matcher` — regex against the tool name (PreToolUse / PostToolUse) or event sub-source (SessionStart `startup|resume|clear|compact`). Empty string `""` matches everything.
- `type` — `command`, `prompt`, `agent`, `http`, `mcp_tool`.
- `command` — shell command for `command` type; URL for `http`; tool name for `mcp_tool`.
- `args` — exec-form args (v2.1.139+).
- `timeout` — seconds; default 600 (10 min).
- `statusMessage` — spinner text shown while the hook runs. **Always set this.**
- `async` — when `true`, hook runs in the background, cannot block, and delivers output on the next conversation turn.

### Interactive setup

Type `/hooks` in Claude Code to open the interactive hooks manager. Lists all active hooks with their source scope (`[User]`, `[Project]`, `[Local]`, `[Plugin: foo]`); lets you add, edit, and delete without hand-editing JSON.

---

## 5. Step-by-Step Creation Process

Walking through a complete `PreToolUse` Bash-logger hook from scratch. The hook logs every Bash command to a JSONL file and does *not* block.

### Step 1 — Decide event and scope

- **Event:** `PreToolUse` (we want to log *before* the command runs).
- **Scope:** User-global (`~/.claude/settings.json`) — applies to every project.
- **Matcher:** `"Bash"` — only Bash tool calls.

### Step 2 — Create the script directory

```bash
mkdir -p ~/.claude/hooks
```

### Step 3 — Write the hook script

`~/.claude/hooks/log-bash.py`:

```python
#!/usr/bin/env python
"""Log every Bash tool call to ~/.claude/logs/bash.jsonl."""
import json
import sys
from datetime import datetime
from pathlib import Path

LOG_DIR = Path.home() / ".claude" / "logs"
LOG_DIR.mkdir(parents=True, exist_ok=True)
LOG_FILE = LOG_DIR / "bash.jsonl"

def main():
    try:
        payload = json.load(sys.stdin)
    except json.JSONDecodeError:
        sys.exit(0)  # Don't block on malformed input

    entry = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "session_id": payload.get("session_id"),
        "command": payload.get("tool_input", {}).get("command"),
        "description": payload.get("tool_input", {}).get("description"),
    }
    with LOG_FILE.open("a", encoding="utf-8") as f:
        f.write(json.dumps(entry) + "\n")
    sys.exit(0)

if __name__ == "__main__":
    main()
```

### Step 4 — Register the hook

`~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python ~/.claude/hooks/log-bash.py",
            "statusMessage": "Logging Bash command...",
            "async": true,
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

(On Windows, replace `python ~/.claude/hooks/log-bash.py` with `C:/Python313/python.exe C:/Users/<user>/.claude/hooks/log-bash.py` — see section 8.)

### Step 5 — Restart Claude Code

**Critical:** hooks are snapshotted at startup. Editing `settings.json` without restarting has no effect.

### Step 6 — Verify

Run a Bash command in Claude Code, then:

```bash
tail -n 1 ~/.claude/logs/bash.jsonl
```

You should see a JSONL entry with timestamp, session ID, and the command.

### Step 7 — Iterate

Type `/hooks` in Claude Code to see the hook listed under `[User]`. The status-line spinner should briefly read "Logging Bash command…" when a Bash call fires.

---

## 6. Common Hook Patterns

Bundled here as recipes — see `claude-code-hooks-best-practices.md` for the full rationale on each pattern.

### 6.1 Block dangerous commands (PreToolUse)

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

BLOCKED=("rm -rf /" "drop database" "DROP TABLE" "format c:" "> /dev/sda" "chmod -R 777")
for pattern in "${BLOCKED[@]}"; do
  if [[ "$COMMAND" == *"$pattern"* ]]; then
    echo "BLOCKED: Dangerous command detected: $pattern" >&2
    exit 2
  fi
done
exit 0
```

Registered as `PreToolUse` with `matcher: "Bash"`.

### 6.2 Protect sensitive files (PreToolUse on Edit/Write)

```bash
#!/bin/bash
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

PROTECTED=(".env" ".env.local" "credentials" "secrets" ".git/config" "id_rsa")
for p in "${PROTECTED[@]}"; do
  [[ "$FILE_PATH" == *"$p"* ]] && echo "Protected file: $FILE_PATH" >&2 && exit 2
done
exit 0
```

Registered as `PreToolUse` with `matcher: "Edit|Write"`.

### 6.3 Auto-format after edits (PostToolUse Prettier)

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write",
            "statusMessage": "Formatting file with Prettier...",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

### 6.4 Inject context on session start

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Project uses Git Bash on Windows. Use Unix commands only. Never auto-commit.'"
          }
        ]
      },
      {
        "matcher": "compact",
        "hooks": [
          {
            "type": "command",
            "command": "cat \"$CLAUDE_PROJECT_DIR/.claude/context-reminder.md\""
          }
        ]
      }
    ]
  }
}
```

### 6.5 Quality gate on Stop (run tests before allowing completion)

```bash
#!/bin/bash
INPUT=$(cat)
STOP_ACTIVE=$(echo "$INPUT" | jq -r '.stop_hook_active // false')

# Prevent infinite loop
if [ "$STOP_ACTIVE" = "true" ]; then
  exit 0
fi

cd "$CLAUDE_PROJECT_DIR"
if ! npm test > /dev/null 2>&1; then
  echo '{"decision":"block","reason":"Tests are failing. Please fix before completing."}'
else
  exit 0
fi
```

### 6.6 Desktop notification (Windows)

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe -Command \"[System.Reflection.Assembly]::LoadWithPartialName('System.Windows.Forms'); [System.Windows.Forms.MessageBox]::Show('Claude Code needs your attention', 'Claude Code')\""
          }
        ]
      }
    ]
  }
}
```

### 6.7 Subagent context injection (SubagentStart)

```json
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "Explore",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Focus on security-related code patterns. Flag any hardcoded credentials.'"
          }
        ]
      }
    ]
  }
}
```

### 6.8 Allowlist auto-approval (PreToolUse, plugin pattern)

The canonical plugin pattern for unblocking background subagents. The hook reads the PreToolUse JSON and emits `permissionDecision: "allow"` for narrowly-defined plugin-internal patterns; everything else falls through to the default permission flow.

```python
#!/usr/bin/env python
import json, re, sys

ALLOWLIST = [
    re.compile(r'python\s+"?\$\{CLAUDE_PLUGIN_ROOT\}/skills/[\w-]+/scripts/[\w-]+\.py"?.*'),
    re.compile(r'dbt\s+(run|test|build|parse|debug|seed|snapshot|compile|deps|list)\b.*'),
    re.compile(r'git\s+(init|status|add|commit|log|show|diff|branch|rev-parse)\b.*'),
]

payload = json.load(sys.stdin)
command = payload.get("tool_input", {}).get("command", "")

if any(p.match(command) for p in ALLOWLIST):
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "allow",
            "permissionDecisionReason": "Plugin-internal command matched allowlist"
        }
    }))
sys.exit(0)
```

Full design rationale in `claude-code-hooks-best-practices.md` § "Plugin-Level Hooks — Patterns from Shipped Plugins (2026)" subsections 2.2 (PreToolUse allowlist for Bash) and 2.3 (allowlist-design rules).

---

## 7. Testing Hooks

### Pipe test JSON to your script before deploying

```bash
# Test the block-dangerous-commands hook
echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf /"},"session_id":"test"}' \
  | python ~/.claude/hooks/block-dangerous.py
echo "Exit code: $?"
```

Build a fixtures folder with sample JSON for each event you care about:

```
~/.claude/hooks/fixtures/
├── pretooluse-bash.json
├── pretooluse-edit.json
├── userpromptsubmit.json
├── sessionstart-startup.json
└── stop.json
```

Then test with `cat fixtures/pretooluse-bash.json | python hook.py`.

### Debug mode

```bash
claude --debug
```

Shows hook execution details in output, including which hook fired and what it returned.

### Verbose mode

Press `Ctrl+O` during a session to toggle verbose output — shows hook progress and stderr from non-blocking errors.

### `/hooks` command

Lists all active hooks with source scope. Quickly confirm a new hook is registered and not shadowed by another scope.

### Manual fixture-driven testing pattern

```python
# test_hooks.py
import json, subprocess
from pathlib import Path

FIXTURES = Path(__file__).parent / "fixtures"

def run_hook(script, fixture_name):
    fixture = (FIXTURES / fixture_name).read_text()
    result = subprocess.run(
        ["python", script],
        input=fixture,
        capture_output=True,
        text=True,
    )
    return result.returncode, result.stdout, result.stderr

# Sanity check: dangerous command is blocked
code, _, err = run_hook("hooks/block-dangerous.py", "pretooluse-bash-rmrf.json")
assert code == 2 and "BLOCKED" in err
```

---

## 8. Cross-Platform Considerations (Windows + Linux/macOS)

### Windows — the #1 gotcha

Hook commands on Windows **do not use Git Bash path translation**, even though Claude Code requires Git Bash to install. Commands must use **Windows-native paths with forward slashes**.

| Path format | Works? | Example |
|-------------|--------|---------|
| Windows native (forward slashes) | **Yes** | `C:/Python313/python.exe C:/Users/<user>/.claude/hooks/script.py` |
| Git Bash `/c/` mount paths | **No** | `/c/Python313/python /c/Users/<user>/.claude/hooks/script.py` |
| Tilde `~` expansion | **No** | `python ~/.claude/hooks/script.py` |
| `$HOME` variable | **No** | `python "$HOME/.claude/hooks/script.py"` |

**Silent failures:**

- Async hooks with wrong path formats fail with **no error output**, no logs, nothing happens.
- Synchronous hooks (without `"async": true`) show a brief `hook error` message in the status line — that's your only diagnostic signal.

**Debugging strategy on Windows:**

1. Start with a synchronous hook so errors are visible.
2. Add a test beacon (`echo test > C:/Users/<user>/hook-test.txt`) to isolate path vs. script issues.
3. Use `python.exe` with the full Windows path.
4. **Restart Claude Code** after editing `settings.json` — hooks are snapshotted at startup.

Working Windows example:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "C:/Python313/python.exe C:/Users/<user>/.claude/hooks/agent-logger.py",
            "statusMessage": "Initializing agent logging..."
          }
        ]
      }
    ]
  }
}
```

### Linux / macOS

Standard POSIX paths work as expected:

```json
"command": "python3 ~/.claude/hooks/log-bash.py"
```

### Cross-platform plugin hooks (works everywhere)

Plugin hooks ship to wherever the user installs the plugin, so they must run on Windows AND Linux/macOS:

- Use `python`, NOT `python3` — Windows has `python.exe`, not `python3`. The Python launcher handles version selection on both platforms.
- Use forward slashes inside `${CLAUDE_PLUGIN_ROOT}` paths — they work on Windows in Python and most shell contexts. Backslashes break Git Bash double-quoted strings.
- Avoid bash-only constructs (`[[ ... ]]`, process substitution `<(...)`, `function() { }`) inside hook scripts. Use Python for any non-trivial logic.
- Don't hardcode `C:/Python313/python.exe` in plugin hooks — that works for global hooks but not for plugins shipped to other machines.

Use `pathlib`, not raw string concatenation:

```python
from pathlib import Path
PLUGIN_ROOT = Path(os.environ["CLAUDE_PLUGIN_ROOT"])
script = PLUGIN_ROOT / "skills" / "data-profiler" / "scripts" / "profile.py"
```

---

## 9. Plugin-Level Hooks (registration in plugin.json)

Plugins can ship hooks at the plugin level. For security, **plugin-shipped agents cannot declare `hooks:`, `mcpServers:`, or `permissionMode:` in their own `agent.md` frontmatter** — those fields are silently stripped at install time. Plugin hooks must live in `plugin.json`.

> *"For security reasons, `hooks`, `mcpServers`, and `permissionMode` are not supported for plugin-shipped agents."* — `code.claude.com/docs/en/plugins-reference`

### plugin.json hooks block

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
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
    ],
    "PreToolUse:write-validator": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/validate-structure.py",
            "statusMessage": "Validating plugin structure..."
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python ${CLAUDE_PLUGIN_ROOT}/hooks/session-config-check.py"
          }
        ]
      }
    ]
  }
}
```

### `${CLAUDE_PLUGIN_ROOT}` — the canonical path placeholder

Plugins must use `${CLAUDE_PLUGIN_ROOT}` (not `$HOME/.claude/skills/...` or hardcoded paths) so the hook works regardless of where the plugin cache lives.

**Known issues to be aware of** (open as of v2.1.150):

- Issue #27145: `CLAUDE_PLUGIN_ROOT` is not set for SessionStart hooks in some configurations.
- Issue #36585: `CLAUDE_PLUGIN_ROOT` is not passed to UserPromptSubmit hooks in the plugin system.
- Issue #38699: `CLAUDE_PLUGIN_ROOT` inconsistent between hooks/skills and agent environment for local marketplace plugins.

Workaround: read `os.environ.get("CLAUDE_PLUGIN_ROOT", os.path.dirname(__file__))` in Python; if the env var is missing, derive the plugin root from `__file__`.

### Three canonical plugin hook patterns

These come from shipped plugins — see `claude-code-hooks-best-practices.md` for full design rationale.

1. **PreToolUse Bash allowlist** — auto-approve narrowly-defined plugin-internal commands so background subagents can run without prompting. Allow-only — never deny. Exit 0 always.
2. **Write/Edit structural validator** — block file writes that violate plugin-specific rules (e.g., block `.py` files under `3 - Notebooks/` in a Fabric plugin).
3. **SessionStart config-check** — read the plugin's options from `~/.claude/settings.json`, detect missing required values, and print a setup-instructions block to stderr. Idempotent: silent when all config is set.

Cross-link: full pattern details and lessons in `claude-code-hooks-best-practices.md` § "Plugin-Level Hooks — Patterns from Shipped Plugins (2026)" and `claude-code-plugin-best-practices.md`.

---

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Hook doesn't fire at all | `settings.json` syntax error (any malformed hook invalidates the whole file as of v2.1.136 — fix only catches some shapes) | Validate JSON with `jq empty < ~/.claude/settings.json` |
| Hook doesn't fire — no syntax error | Forgot to restart Claude Code after editing | Restart. Hooks are snapshotted at startup. |
| Hook fires but no visible effect | Wrong matcher (e.g., `"bash"` lowercase vs. `"Bash"`) | Matchers are regex against the tool name; check case |
| Hook blocks unexpectedly with no message | Exit 2 with empty stderr | Always emit a stderr message before `exit 2` |
| `permissionDecision: "deny"` ignored | User has explicit `permissions.allow` rule that takes precedence | Hooks can't override explicit user permissions — feature, not bug |
| `permissionDecision: "allow"` ignored | User has explicit `permissions.deny` rule | Same — user deny rules always win |
| Background subagent says "I am unable to execute Bash commands" | No PreToolUse allowlist hook; `acceptEdits` only covers filesystem Bash | Ship a PreToolUse allowlist hook (section 6.8) |
| Async hook fails silently | No way to see errors for async hooks | Switch `"async": false` temporarily to surface errors in status line |
| Hook works locally, breaks for other users on plugin install | Hardcoded path (`C:/Python313/python.exe`) | Use `python` (no path) and `${CLAUDE_PLUGIN_ROOT}` |
| Hook fires twice for the same event | You have the same hook in two scopes (user + plugin, or project + local) | Claude Code deduplicates identical commands, but different commands run additively |
| `CLAUDE_PLUGIN_ROOT` not set inside hook | Known open issue (#27145, #36585, #38699) for some event types | Derive from `__file__` as a fallback in Python |
| WorktreeRemove hook destroys untracked notebooks / data | The hook runs `git worktree remove --force` which wipes untracked files | See `claude-code-hooks-best-practices.md` § "Worktree hooks can silently destroy agent output" |
| `prompt`-type or `agent`-type hooks fail to bind to SessionStart | Hook handler type not supported for that event (clearer error msg added in v2.1.142) | Use `type: "command"` for SessionStart / Setup / SubagentStart |
| Hook output not visible to Claude | `suppressOutput: true` in hook output | Set `false` or omit |
| Hook never exits | Hook is reading stdin in a way that blocks | Always wrap `json.load(sys.stdin)` in a try/except and exit 0 on error |

### When all else fails

1. Add a synchronous test beacon: `"command": "echo test > /tmp/hook-test.txt"` (or `C:/Users/<user>/hook-test.txt` on Windows). If the file appears, the hook is firing — your script logic is the issue. If not, the registration or path format is wrong.
2. Run `claude --debug` and watch the hook execution log.
3. Check `/hooks` output — confirms which scope the hook is loaded from and that the matcher is parsed correctly.

---

## Further Reading

- **Companion docs in this folder:**
  - `claude-code-hooks-best-practices.md` — patterns, rationale, plugin-level lessons
  - `claude-code-plugin-best-practices.md` — broader plugin packaging
  - `how-to-create-plugins.md` — step-by-step plugin guide
  - `how-to-create-agents.md`, `how-to-create-skills.md` — sibling how-tos
- **Official Anthropic docs:**
  - `code.claude.com/docs/en/hooks` — hooks reference
  - `code.claude.com/docs/en/hooks-guide` — hooks guide
  - `code.claude.com/docs/en/plugins-reference` — plugin schema including hooks
  - `code.claude.com/docs/en/best-practices` — hooks section in best practices
  - `claude.com/blog/how-to-configure-hooks` — Anthropic blog post
- **Community reference implementations:**
  - `github.com/disler/claude-code-hooks-mastery` — IndyDevDan's reference repo
  - `github.com/disler/claude-code-hooks-multi-agent-observability` — hooks → WebSocket → dashboard pattern
  - `github.com/disler/claude-code-damage-control` — defense-in-depth PreToolUse blocks
  - `baugues.com/hooks` — Greg Baugues's single-file Python tutorial
- **Conference / video coverage:** see § 14 of `claude-code-hooks-best-practices.md`.
