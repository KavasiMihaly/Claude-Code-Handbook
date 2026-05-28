# Claude Code ↔ GitHub Copilot — Primitive Mapping

**Last Updated**: May 2026
**Purpose**: For each Claude Code building block — agents, skills, hooks, plugins, MCP, memory, the CLI — the GitHub Copilot equivalent: its **name**, what it **does**, and its exact **file format**. So you can carry a mental model (or actual files) from one to the other.

---

## Table of Contents

1. [The Short Version](#the-short-version)
2. [A Note on Moving Targets](#a-note-on-moving-targets)
3. [Project Memory — `CLAUDE.md`](#project-memory--claudemd)
4. [Subagents](#subagents) · [Agent-invokes-agent (orchestrator pattern)](#agent-invokes-agent--the-orchestrator-pattern)
5. [Skills](#skills)
6. [Hooks](#hooks)
7. [Slash Commands](#slash-commands)
8. [Plugins & Marketplace](#plugins--marketplace)
9. [MCP Servers](#mcp-servers)
10. [The CLI](#the-cli)
11. [Cross-Compatibility — Copilot Reads Claude's Files](#cross-compatibility--copilot-reads-claudes-files)
12. [Caveats & Preview Flags](#caveats--preview-flags)

---

## The Short Version

| Claude Code | GitHub Copilot equivalent | Copilot file / location | Maturity |
|---|---|---|---|
| `CLAUDE.md` (project memory) | Repository custom instructions / `AGENTS.md` | `.github/copilot-instructions.md`, `AGENTS.md` (root + nested) | GA, broadest support |
| Path-scoped memory | Scoped instruction files | `.github/instructions/*.instructions.md` (`applyTo` glob) | GA |
| Subagents (`agent.md`) | Custom agents (was "custom chat modes") | `.github/agents/*.agent.md` (was `.chatmode.md`) | GA-ish; editor UI in preview |
| Orchestrator calling `Agent()` | `runSubagent` tool + `agents` allowlist (VS Code); agents-as-tools (CLI) — model-driven, not declarative; **no cloud equivalent** | `.agent.md` frontmatter `tools: ['runSubagent']`, `agents: [...]` | VS Code / CLI only |
| Autonomous / background agent | Copilot **coding agent** ("cloud agent") | runs in Actions; setup via `.github/workflows/copilot-setup-steps.yml` | GA |
| Skills (`SKILL.md`) | **Agent Skills** (`SKILL.md` — same open standard) | `.github/skills/`, `.claude/skills/`, `~/.copilot/skills/` | New (Dec 2025) |
| Hooks (PreToolUse, …) | Copilot **hooks** (near 1:1, reads Claude's format) | `.github/hooks/*.json`, `~/.copilot/hooks/` | CLI/cloud GA; **VS Code preview** |
| Slash commands | **Prompt files** | `.github/prompts/*.prompt.md` | GA (editors only) |
| Plugins + marketplace | **Copilot Extensions** (GitHub Apps) → GitHub Marketplace; Agent Skills via `gh skill` | App settings UI; git repos | GA |
| MCP servers (`.mcp.json`) | MCP config (different root key) | `.vscode/mcp.json` (key `servers`); coding-agent config (key `mcpServers`) | GA |
| Claude Code CLI | **GitHub Copilot CLI** (`copilot`) | reads AGENTS.md, agents, skills, hooks, MCP | GA |

**Headline takeaways:**
- Copilot has **converged hard** on Claude Code's model in the last few months — there's now a near-1:1 equivalent for *every* primitive, and Copilot deliberately **reads Claude Code's files** (`.claude/skills/`, `CLAUDE.md`, `.claude/settings.json` hooks).
- The biggest naming churn: Copilot's **"custom chat modes" (`.chatmode.md`) are being renamed to "custom agents" (`.agent.md`)**, and the prompt-file `mode:` frontmatter key is becoming `agent:`.
- Copilot splits into **two surfaces with different rules**: the **editors** (VS Code / Visual Studio / JetBrains) and **github.com / the cloud agent**. Prompt files and custom agents are editor-only; instructions and `AGENTS.md` work everywhere.

---

## A Note on Moving Targets

Copilot's customization layer shipped most of these features in late 2025 and is still renaming and consolidating them. Where a feature is mid-rename or in preview, this doc names both forms and flags it. Verify exact frontmatter keys against current docs before relying on them — several shipped within the last few months. (Sources are GitHub Docs, the GitHub Changelog, and the VS Code Copilot docs; URLs in [Caveats](#caveats--preview-flags).)

---

## Project Memory — `CLAUDE.md`

**Claude Code:** `CLAUDE.md` (project root, nested, and `~/.claude/CLAUDE.md` for global) auto-loads as standing context every session.

**Copilot equivalent — three layers:**

| Copilot feature | File / path | Applies | Notes |
|---|---|---|---|
| Repository custom instructions | `.github/copilot-instructions.md` | Always-on, repo-wide | No frontmatter. Broadest support (editors, coding agent, code review, github.com). Code review reads only first ~4,000 chars. |
| `AGENTS.md` | `AGENTS.md` root + nested in subfolders | Always-on | If both this and `copilot-instructions.md` exist, **both are used**. Nearest nested file wins in its subtree. |
| Scoped instructions | `.github/instructions/*.instructions.md` | Only files matching `applyTo` glob | Frontmatter: `applyTo` (comma-separated globs), `description`, `name`, `excludeAgent`. Body can reference `#tool:<name>`. |

**Mapping note:** Copilot recognizes `AGENTS.md`, **`CLAUDE.md`**, and `GEMINI.md` as root agent-instruction files (`CLAUDE.md` uses a `paths` array where `.instructions.md` uses `applyTo`). github.com precedence when sources conflict: personal > path-specific > repo-wide > `AGENTS.md` > organization.

---

## Subagents

**Claude Code:** subagents defined in `agent.md` (frontmatter: `name`, `description`, `tools`, `model`), invoked by the Task tool or `/agents`; plus background/remote autonomous agents.

**Copilot — two distinct things:**

**1. Custom agents** *(the direct subagent analog)* — a persistent persona bundling a model, a restricted tool set, and behavior.
- **File:** `.github/agents/*.agent.md` (project) or `~/.copilot/agents/` (user). **Legacy name:** "custom chat modes," extension `.chatmode.md` in `.github/chatmodes/` — VS Code now says *rename `.chatmode.md` → `.agent.md`*.
- **Frontmatter:** `name`, `description`, `tools`, `model`, and `handoffs` (a UI agent-*switch*, not a subagent call — see [the orchestrator section](#agent-invokes-agent--the-orchestrator-pattern) for why these are different).
- **Invoked:** agents dropdown in Chat, `/agent`, by name/inference, or `copilot --agent <name> --prompt "…"`.

**2. Copilot coding agent** *(the autonomous/background analog)* — works a GitHub issue/PR on its own.
- Triggered by assigning an issue to Copilot, `@copilot`, or delegating from the IDE/Agents page.
- Runs in an **ephemeral GitHub Actions environment**, explores the repo, runs tests/linters, opens a **draft PR**.
- Environment setup file: **`.github/workflows/copilot-setup-steps.yml`** — the job **must** be named `copilot-setup-steps`; only `steps`, `permissions`, `runs-on`, `services`, `snapshot`, `timeout-minutes` (≤59) are honored.

### Agent-invokes-agent — the orchestrator pattern

This is the pattern the OneDayBI plugins lean on: an **orchestrator agent** (e.g. `dbt-pipeline-orchestrator`, `fabric-migration-orchestrator`) calls the `Agent`/`Task` tool to spawn specialized subagents (`business-analyst`, `data-explorer`, the staging/dim/fact builders), runs them — sometimes in parallel — collects their results, and drives the next stage.

**Does Copilot do this? Yes — but only in VS Code agent mode and the Copilot CLI, not the cloud coding agent.** The trap is that Copilot has *three* "multiple agents" mechanisms and only two are real orchestration:

| Copilot mechanism | What it actually is | Orchestration? | Claude Code analog |
|---|---|---|---|
| **`runSubagent` tool + `agents` allowlist** (VS Code); custom-agents-as-tools (CLI) | Parent calls a named subagent, which runs in **isolated context** and **returns a summary**; parent then continues | **Yes — call-and-return** | Orchestrator agent calling `Agent()` to spawn a subagent |
| **`/fleet`** (CLI) | Orchestrator decomposes one objective into **different** parallel subtasks, dispatches them (can target specialized custom agents via `@name`), polls, converges | **Yes — parallel, but user-invoked only** | Parallel Task fan-out / agent teams |
| **`handoffs`** frontmatter (`.agent.md`) | A post-response **button** that switches the user to another agent with a pre-filled prompt | **No — one-way agent *switch*, no return** | (none — it's UI navigation, not a subagent call) |

**The real analog — VS Code `runSubagent`.** Enable the `runSubagent` tool and declare an `agents` allowlist; the model invokes subagents automatically (or the user types `#runSubagent`). Each gets isolated context and returns a summary:

```yaml
---
name: orchestrator
description: Coordinates planning, implementation, and review.
tools: ['runSubagent', ...]
agents: [planner, implementer, reviewer]   # allowlist of callable subagents
---
```

In the **Copilot CLI** it's implicit: *"your custom agents are made available as tools to Copilot, and the model will start a new agentic loop using a relevant custom agent when necessary"* — agents-as-tools, the same shape as Claude Code's Task tool.

**Where it differs from the plugin pattern — read before assuming parity:**

1. **Model-driven, not a declarative workflow.** Claude Code orchestrator agents (and `Workflow` scripts) let you *spell out* "run A, then B and C in parallel, then D, collecting structured output at each step." Copilot has **no manifest for that sequence** — decomposition is decided by the model at runtime. The closest declarative artifact is the `agents` allowlist plus prose in the `.agent.md` body; `/fleet`'s wave-planning is also runtime model logic.
2. **`/fleet` can't be fired programmatically** — it's a user-typed slash command, not something an agent triggers mid-run. Fleet subagents **share the filesystem**, get isolated context, and **can't message each other** — only the orchestrator coordinates (via a per-session SQLite TODO graph).
3. **The cloud coding agent (github.com) has no internal subagent spawning.** It ignores `handoffs`; "multiple agents" there means *human-started concurrent sessions* in the Agents panel, plus cross-surface `/delegate` (CLI → cloud: one task to one autonomous agent). A true server-side orchestrator is an open feature request, not a shipped capability.
4. **Nesting:** off by default in both tools (Claude Code subagents also can't spawn subagents) — but VS Code can **opt in** to nested subagents up to depth 5 (`chat.subagents.allowInvocationsFromSubagents`), which Claude Code subagents don't expose.

> **Porting a plugin orchestrator:** the *capability* exists in VS Code agent mode / Copilot CLI — write an orchestrator `.agent.md` with `runSubagent` + an `agents` allowlist, and author your `business-analyst` / `data-explorer` / builder agents as callable subagents. What you **lose** is the explicit, repeatable workflow contract: stage ordering and parallelism become instructions for the model to follow, not a guaranteed sequence. And there is **no github.com-side orchestrator** at all — that whole tier of your plugins has no cloud equivalent yet.

---

## Skills

**Claude Code:** a Skill is a folder with `SKILL.md` (YAML frontmatter `name` + `description`, progressive-disclosure body, optional `scripts/`, `references/`, `assets/`), auto-loaded when the description matches the task.

**Copilot equivalent — Agent Skills** *(new, December 2025 — same open standard as Anthropic Skills)*:
- **File:** **`SKILL.md`** — Markdown + YAML frontmatter, identical concept.
- **Required frontmatter:** `name` (lowercase/hyphens, ≤64 chars, must match the parent directory), `description` (≤1024 chars).
- **Optional:** `argument-hint`, `user-invocable` (default true), `disable-model-invocation` (default false), `context` (`inline` | `fork` — fork experimental).
- **Project locations:** `.github/skills/`, **`.claude/skills/`**, `.agents/skills/`. **Personal:** `~/.copilot/skills/`, **`~/.claude/skills/`**, `~/.agents/skills/`.
- **Surfaces:** coding agent, Copilot CLI, VS Code agent mode (Insiders). Forked context needs `github.copilot.chat.skillTool.enabled`.
- **Discovery:** `gh skill` CLI; sources include `anthropics/skills` and `github/awesome-copilot`.

> Because Copilot reads `.claude/skills/`, a Claude Code skill folder is **partly portable to Copilot as-is**. This is the closest cross-tool primitive of the bunch.

**Watch the terminology trap:** a Copilot Extension **"skillset"** is *not* the same thing — that's a remote API-endpoint definition (see [Plugins & Marketplace](#plugins--marketplace)). The local-folder analog is **Agent Skills**.

---

## Hooks

**Claude Code:** shell/HTTP commands fired on lifecycle events (`PreToolUse`, `PostToolUse`, `SessionStart`, `Stop`, …), registered in `settings.json` or a plugin's `hooks/`.

**Copilot equivalent — Copilot hooks** *(near 1:1; deliberately interoperates with Claude's format)*:
- **File:** `NAME.json` in **`.github/hooks/`** (repo) or `~/.copilot/hooks/` (user). Copilot **also reads** Claude-style `.claude/settings.json` / `settings.local.json` hook blocks.
- **Shape:** `{ "version": 1, "disableAllHooks": false, "hooks": { "preToolUse": [...], "postToolUse": [...] } }`.
- **Events** (both camelCase CLI and PascalCase VS Code forms accepted): `sessionStart`/`SessionStart`, `sessionEnd`, `userPromptSubmitted`/`UserPromptSubmit`, `preToolUse`/`PreToolUse`, `postToolUse`, `postToolUseFailure`, `errorOccurred`, `preCompact`, `agentStop`/`Stop`, `subagentStart`, `subagentStop`, plus CLI-only `permissionRequest`, `notification`.
- **Entry types:** command (`bash`/`powershell`/`command`, with `cwd`, `env`, `timeoutSec` default 30), HTTP (`url` POST + `headers` + `allowedEnvVars`), and CLI-only prompt hooks.
- **Decision control:** `preToolUse` returns `{ "permissionDecision": "allow|deny|ask", "permissionDecisionReason": "…", "modifiedArgs": {…} }` — approve, deny, or rewrite tool args. Maps almost exactly to Claude Code's PreToolUse. A `matcher` regex filters by tool name. Cloud agent treats `"ask"` as `"deny"` (non-interactive).

This is the most surprising convergence: a year ago Copilot had no hooks. Now it ships an event model that mirrors Claude Code's and reads Claude's own hook config.

---

## Slash Commands

**Claude Code:** custom slash commands (and skill-backed `/commands`) for reusable, manually-invoked prompts.

**Copilot equivalent — Prompt files:**
- **File:** `*.prompt.md` in `.github/prompts/` (workspace) or VS Code user profile.
- **Invoked:** type `/` + the prompt name in Chat (slash-command style), the **Chat: Run Prompt** command, or the editor run button. **Manually invoked**, not auto-applied.
- **Frontmatter:** `description`, `name` (the slash name; defaults to filename), `argument-hint`, **`agent`** (replaces the older `mode:` — values `ask`/`agent`/`plan`/custom-agent name), `model`, `tools`.
- **Variables:** `${input:variableName}` / `${input:variableName:placeholder}`; workspace/selection variables (`${selection}`, `${file}`, `${workspaceFolder}`, …) and `#tool:<name>` references.
- **Surface:** editors only (VS Code / Visual Studio / JetBrains) — **not on github.com**.

---

## Plugins & Marketplace

**Claude Code:** a **plugin** bundles skills + agents + hooks + MCP servers + commands (`.claude-plugin/plugin.json`); distributed through a **marketplace** (`.claude-plugin/marketplace.json`, git-hosted) and installed via `/plugin`.

**Copilot — two parallel channels (it does *not* bundle the way plugins do):**

**1. Copilot Extensions** — built as **GitHub Apps**, published to the **GitHub Marketplace** (public or org-private). Two types:
- **Agents** — a GitHub App with your own hosted backend; full control over request/response; built with the Copilot Extensions SDK. Heavier than a Claude skill — a server-side integration.
- **Skillsets** — lightweight: **up to 5 API endpoints** ("skills") Copilot calls directly. Each registered in the **GitHub App settings UI** (not a file): `name`, `inference_description`, `url`, `parameters` (JSON Schema), `return_type`. Copilot handles routing/prompting automatically.

**2. Agent Skills via `gh skill`** — git-repo / CLI-based sharing (e.g. `github/awesome-copilot`, `anthropics/skills`). This is the model **closest to Claude Code plugin marketplaces** (git-hosted, file-based).

So Claude Code consolidates everything into one git-hosted **plugin**; Copilot spreads it across **App-Marketplace Extensions** (agents/skillsets) and **repo-based Agent Skills**.

---

## MCP Servers

Both support MCP — but the **config file and root key differ**, which is the gotcha when porting.

| | Claude Code | Copilot (VS Code) | Copilot coding agent (github.com) |
|---|---|---|---|
| File | `.mcp.json` (repo) | `.vscode/mcp.json` | Repo Settings → Copilot → coding agent (JSON in UI) |
| **Root key** | `mcpServers` | **`servers`** | `mcpServers` |
| Transports | `stdio` / `http` / `sse` | `stdio` (default) / `http` / `sse` | `local`/`stdio` / `http` / `sse` |
| Per-server fields | `command`, `args`, `env`, `type`, `url` | + `headers`, `sandbox`, `${input:…}` | **`tools` array required** (`["*"]` for all); secrets via `COPILOT_MCP_`-prefixed env |

```jsonc
// VS Code: .vscode/mcp.json   (note the "servers" key)
{ "servers": {
    "github":     { "type": "http", "url": "https://api.githubcopilot.com/mcp" },
    "playwright": { "command": "npx", "args": ["-y", "@microsoft/mcp-server-playwright"] }
} }
```

```jsonc
// Copilot coding agent   (note "mcpServers" + required "tools")
{ "mcpServers": {
    "sentry": {
      "type": "local",
      "command": "npx",
      "args": ["@sentry/mcp-server@latest"],
      "tools": ["*"],
      "env": { "SENTRY_ACCESS_TOKEN": "$COPILOT_MCP_SENTRY_ACCESS_TOKEN" }
} } }
```

Porting a Claude Code `.mcp.json` to VS Code = rename the root key `mcpServers` → `servers`. Porting to the coding agent = keep `mcpServers`, but add a `tools` array to each server.

---

## The CLI

**Claude Code:** the `claude` CLI — interactive agent, slash commands, reads `CLAUDE.md`, runs subagents/skills/hooks/MCP.

**Copilot equivalent — GitHub Copilot CLI** (`copilot`; since `gh` v2.86.0, `gh copilot` installs/runs it):
- Agentic terminal assistant — multi-step workflows, file edits, shell commands (with approval), GitHub.com interaction. Modes: ask / edit / agent. Parallel subagents via `/fleet`.
- **Familiar slash commands:** `/model`, `/context`, `/compact`, `/mcp`, `/agent`, `/allow-all`, `/yolo`, `/feedback`.
- **Reads:** `AGENTS.md` (root + nested), `.github/copilot-instructions.md`, `.github/instructions/**/*.instructions.md`, custom agents (`*.agent.md`), Agent Skills (`SKILL.md`), hooks, MCP, plus "Copilot Memory."

> **Don't confuse it with the old tool.** The deprecated `gh-copilot` extension (`gh copilot suggest`/`explain`) was suggestion-only and **stopped functioning Oct 25, 2025**. The Claude Code CLI analog is the new agentic **`copilot`**, not the old extension.

---

## Cross-Compatibility — Copilot Reads Claude's Files

Copilot has intentionally built interop with Claude Code's on-disk format. If you already run Claude Code, these carry over with little or no change:

| Claude Code file | Read by Copilot? |
|---|---|
| `CLAUDE.md` (root) | Yes — recognized as a root agent-instruction file (toggle `chat.useClaudeMdFile`). |
| `.claude/skills/` (and `~/.claude/skills/`) | Yes — a valid Agent Skills location. |
| `.claude/settings.json` hooks (and `settings.local.json`) | Yes — Copilot hooks read Claude's hook blocks. |
| `~/.claude/rules`, `.claude/rules` | Yes — read as instruction sources. |

The reverse (Claude Code reading Copilot's `.github/prompts/` or `.chatmode.md`) is **not** supported.

---

## Caveats & Preview Flags

- **Renames in flight:** `.chatmode.md` → `.agent.md`; prompt-file `mode:` → `agent:`. Old files still work but VS Code recommends migrating.
- **Preview / experimental:** VS Code agent hooks (format may change); the Agent Customizations editor; skill `context: fork`; nested `AGENTS.md` (`chat.useNestedAgentsMdFiles`, plus a known VS Code handling bug). Org/enterprise-level Agent Skills "coming soon."
- **Editor-only vs everywhere:** prompt files and custom agents are editor-only; instructions and `AGENTS.md` have the broadest cross-surface reach. Code review reads only the first ~4,000 chars of instruction files.
- **Subagent orchestration is young and surface-limited:** `runSubagent` / agents-as-tools work in VS Code agent mode and the Copilot CLI only — not the github.com cloud agent — and are evolving fast (open VS Code issues on named-subagent execution and per-subagent models). Custom agents are public preview on JetBrains/Eclipse/Xcode.
- **Verify before relying:** much of this shipped within months of writing — confirm exact frontmatter/JSON keys against current docs.

**Primary sources:**
`docs.github.com/copilot/...` (custom instructions, response customization, hooks-reference, agent skills, coding-agent MCP) · `code.visualstudio.com/docs/copilot/customization/...` (custom-instructions, prompt-files, custom-chat-modes, agent-skills, hooks, mcp-servers) · GitHub Changelog entries 2025-07-23 (`.instructions.md`), 2025-08-28 (`AGENTS.md`), 2025-12-18 (Agent Skills GA), 2025-09-25 (`gh-copilot` deprecation).

---

*Part of the [Claude Code Handbook](README.md). Copilot's customization layer is evolving fast — treat naming and preview flags here as a snapshot, not a contract.*
