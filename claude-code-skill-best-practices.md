# Claude Code Skill Best Practices

**Last Updated:** 2026-05-25
**Sources:** Anthropic official documentation, Claude Code release notes (v2.1.111 — v2.1.150), Anthropic engineering blog (3 primary-source skill posts), expert YouTube creators (IndyDevDan, Sam Witteveen, AICodeKing, Thariq Shihipar, Anthropic team), plus production-plugin lessons from `dbt-pipeline-toolkit` and `fabric-dataflow-migration-toolkit`.

> **Companion documents:**
> - `how-to-create-skills.md` — step-by-step build guide
> - `how-to-create-agents.md` — step-by-step agent guide
> - `how-to-create-hooks.md` — step-by-step hook guide
> - `how-to-create-plugins.md` — step-by-step plugin guide
> - `claude-code-agent-best-practices.md` — agent patterns
> - `claude-code-hooks-best-practices.md` — hook patterns
> - `claude-code-plugin-best-practices.md` — plugin patterns

---

## 1. What Are Skills?

**Skills** are modular, self-contained packages that extend Claude's capabilities with specialized workflows, tool integrations, and domain expertise. They are **portable** across Claude Code, the Claude apps (web/mobile), and the Anthropic API.

Anthropic's engineering team defines them this way (verbatim):

> *"Skills extend Claude's capabilities by packaging your expertise into composable resources for Claude, transforming general-purpose agents into specialized agents that fit your needs."*
> — [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills), 2025-10-16

> *"A skill is a directory that contains a `SKILL.md` file."*

> *"Skills are folders that include instructions, scripts, and resources that Claude can load when needed."*
> — [Introducing Agent Skills](https://www.anthropic.com/news/skills), 2025-10-16

> *"Think of Skills as custom onboarding materials that let you package expertise, making Claude a specialist on what matters most to you."*
> — Introducing Agent Skills

The launch on **2025-10-16** introduced the `skills-2025-10-02` beta in the API alongside Anthropic-managed Skills for `.pptx`, `.xlsx`, `.docx`, and `.pdf`. On **2025-12-18** Anthropic published Agent Skills as an [open standard](https://agentskills.io) for cross-platform portability. (Changelog v2.1.142 added flat-layout plugin skills; v2.1.145 added `claude plugin validate` skill validation.)

---

## 2. Skills vs Agents vs Subagents vs MCP vs Prompts

| Mechanism | Purpose | When to Use |
|-----------|---------|-------------|
| **Skills** | Portable, reusable task packs (load on demand) | Repeatable workflows; "teach Claude how" |
| **MCP** | Always-on tool surfaces to external systems | Live integrations with governance |
| **Agents** | AI personas with behavior patterns + skills bundled in `skills:` frontmatter | Specialized roles, persistent identity |
| **Subagents** | Isolated parallel contexts | Verbose work, tool restriction, context separation |
| **Prompts** | One-shot instructions | Non-repeatable tasks |

**Rule of thumb** (paraphrased from the Boris Cherny / Pragmatic Engineer interview):
- Skills are pulled **on demand**, MCP servers are **always on** — choose by call frequency.
- Default to skills first; escalate to MCP only when you need a stateful or external integration.

| Dimension | Skill | Subagent | Agent Team |
|-----------|-------|----------|------------|
| **Context** | Main conversation (unless `context: fork`) | Own isolated window | Own fully independent windows |
| **Communication** | Direct tool use inline | Returns one string to caller | Teammates `SendMessage` each other |
| **Invocation** | User + Claude (auto-discovery) | Claude delegates, `@` mention, `--agent` | Lead spawns via `team_name` |
| **Token cost** | Low (progressive disclosure) | Medium (fresh window) | High (N full instances) |
| **Size target** | SKILL.md <500 lines | agent.md 250-500 lines | Same as subagents |
| **Portability** | Cross-platform (Code/API/Apps) | Claude Code specific | Claude Code specific (experimental) |
| **Best for** | Reusable expertise, workflows, knowledge | Verbose/isolated focused work | Parallel exploration, cross-layer work |
| **Can spawn subagents?** | Via `context: fork` (one-shot) | No | Only lead can |

Source: [claude.com/blog/skills-explained](https://claude.com/blog/skills-explained)

---

## 3. File Structure

```
skill-name/
├── SKILL.md              # REQUIRED — only required file, must be at folder root
├── scripts/              # Optional — executable code (Python, Bash, Node)
│   ├── helper.py
│   └── runner.sh
├── references/           # Optional — Level-3 docs loaded on demand
│   ├── api-reference.md
│   └── examples.md
└── assets/               # Optional — templates, images
    └── template.json
```

- `SKILL.md` is the **only required file**; everything else is optional.
- Skill folder names are lowercase-hyphenated (`dbt-runner`, `sql-formatter`).
- As of **Claude Code v2.1.142**, a plugin with a **root-level `SKILL.md` and no `skills/` subdirectory** is also surfaced as a skill — flat-layout skill-plugins are now supported. (Changelog v2.1.142: *"Plugins with root-level SKILL.md and no skills/ subdirectory now surfaced as skill."*)

---

## 4. Progressive Disclosure (Critical)

> *"Progressive disclosure is the core design principle that makes Agent Skills flexible and scalable."*
> — [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

Skills use a **three-level loading system** so context only fills with what's actively needed:

| Level | What loads | Cost | When |
|-------|------------|------|------|
| **1 — Metadata** | `name` + `description` from YAML frontmatter | ~100 tokens per skill | Always loaded at startup; drives routing |
| **2 — SKILL.md body** | Full markdown body | Up to ~5K tokens | When Claude decides the skill is relevant |
| **3 — Bundled resources** | `scripts/`, `references/`, `assets/` | Variable; effectively unlimited | On demand — scripts execute **without** loading their content into context |

The Level-1 `description` is the single most load-bearing field in the entire skill. It is a **routing signal**, not documentation — the loader uses it to decide whether to expand the skill. (Cross-validated by *"Anthropic's Full Claude Skills Guide In 22 Minutes"* and *"Claude Code's Creator Uses These Skills"* — both creators call out description-as-routing.)

---

## 5. SKILL.md Anatomy & Frontmatter

### Minimal frontmatter (required)

```yaml
---
name: skill-name-here
description: Clear description of what this skill does and when to use it. Include triggers and contexts.
---
```

### Full frontmatter reference (2026)

All fields besides `description` are optional. The schema below reflects what `claude plugin validate` accepts as of **v2.1.145** (when skill validation was tightened to flag `skills:` entries pointing at files instead of directories).

```yaml
---
name: my-skill                         # Optional (defaults to dir name); lowercase-hyphen, ≤64 chars
description: Does X when Y happens...  # REQUIRED; ≤1024 chars, front-load keywords
argument-hint: "[issue-number]"        # Autocomplete hint shown when user types the slash command
disable-model-invocation: false        # true = user-only (e.g., /deploy); Claude cannot auto-trigger
user-invocable: true                   # false = Claude-only background knowledge (not shown in /)
allowed-tools: Read Grep Glob          # Tools Claude may use within this skill without re-prompting
model: sonnet                          # Override active model for the skill run
effort: high                           # low|medium|high|max|xhigh (max = Opus 4.6+; xhigh = Opus 4.7+)
context: fork                          # Run skill body as a forked subagent task (isolated context)
agent: Explore                         # Which subagent type to fork to (when context: fork)
hooks:                                 # Lifecycle hooks during skill execution
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate.sh"
paths: "src/**/*.ts, lib/**/*.ts"      # Auto-activate when matching files are in scope
shell: bash                            # bash|powershell — language for !`command` injection blocks
---
```

### `name` field rules

- Max 64 characters, lowercase letters/numbers/hyphens only, no spaces/underscores/XML tags.
- Examples: `dbt-runner`, `sql-formatter`, `code-reviewer`.

### `description` field rules (most critical)

- Max 1024 characters, non-empty, no XML tags.
- Front-load the **trigger verbs and contexts** — this is what the loader matches against the user's request.

❌ Bad (too vague): `description: Runs dbt commands`

✅ Good (specific triggers and context):
```yaml
description: Execute dbt commands (run, test, compile, docs, build, snapshot, seed, source freshness, deps, debug, list, clean) for analytics engineering workflows. Use when running dbt models, testing data quality, generating documentation, managing dbt projects, running snapshots, or implementing Slim CI patterns. Supports advanced model selection, full refresh, state comparison, and parallel execution.
```

### Dynamic context injection: `` !`<command>` ``

Use `` !`<command>` `` inside SKILL.md to run shell commands **before** the skill is sent to Claude — the output replaces the placeholder. Claude sees the rendered data, not the command itself. Control the shell via the `shell: bash` or `shell: powershell` frontmatter field.

```markdown
## PR context
- Diff: !`gh pr diff`
- Review comments: !`gh pr view --comments`
- Current branch: !`git branch --show-current`
```

**Note (Changelog v2.1.119):** *"Fixed pre-compaction skill re-execution"* — skills no longer re-run after compaction. **v2.1.120:** *"Skills reference effort with `${CLAUDE_EFFORT}`"* — skills can interpolate the current effort level.

### `context: fork` — running a skill in a subagent

A skill can execute inside its own forked subagent context — useful for research/exploration that would otherwise pollute the main conversation with intermediate output.

```yaml
---
name: deep-research
description: Research a topic thoroughly and return a structured summary
context: fork
agent: Explore
---

Research $ARGUMENTS. Find relevant files, read code, synthesize findings.
Return a concise summary (max 500 words) with file paths and line numbers.
```

Use when the skill is read-only, output would consume >10K tokens but the summary is compact, and no interactive input is needed.

### SKILL.md body — size targets

| Category | Lines | Words | Status |
|----------|-------|-------|--------|
| **Optimal** | <200 | <2k | Excellent |
| **Good** | 200-500 | 2k-5k | Good |
| **Acceptable** | 500-800 | 5k-8k | Needs optimization |
| **Too Large** | >800 | >8k | Split or externalize |

The standard fix for an oversized SKILL.md is to push detail into `references/` and link from the body. Anthropic's own internal skills lean on short, structured headings rather than walls of prose. (*"Claude Code's Creator Uses These Skills"* — internal teams use headings/sections; Claude reads structure faster than prose.)

---

## 6. Skill Patterns from Shipped Plugins (2026)

The `dbt-pipeline-toolkit` and `fabric-dataflow-migration-toolkit` plugins surfaced a class of skill-authoring gotchas that don't appear when you author standalone skills in `~/.claude/skills/`. Skills that ship inside a plugin are loaded from a different filesystem location, run inside a constrained subprocess environment, and integrate with plugin-level hooks and `userConfig` blocks. Everything below comes from production breakage on fresh installs — not theory.

See `claude-code-plugin-best-practices.md` for the plugin-author perspective and `how-to-create-plugins.md` for the step-by-step authoring guide.

### 6.1 Use `${CLAUDE_PLUGIN_ROOT}` / `${CLAUDE_SKILL_DIR}` — not `$HOME/.claude/skills/`

Plugin-installed skills do **not** live in `~/.claude/skills/`. Claude Code copies the plugin to a cache directory:

```
~/.claude/plugins/cache/<id>/skills/<skill-name>/
```

Every `python "$HOME/.claude/skills/foo/scripts/bar.py"` invocation silently fails on a fresh install with "file not found", and in background subagents the error never propagates. The dbt plugin shipped with 183 occurrences of this pattern across 11 files and the entire pipeline appeared to run but produced nothing. *(Source: DBT F7, Fabric inherited F7)*

Inside **SKILL.md**, prefer the skill-scoped variable:

```bash
python "${CLAUDE_SKILL_DIR}/scripts/profile_data.py" --file customers.csv
```

Inside an **agent.md** referencing a sibling skill, use the plugin-root variable:

```bash
python "${CLAUDE_PLUGIN_ROOT}/skills/data-profiler/scripts/profile_data.py" --file customers.csv
```

Both get substituted inline at load time — the docs say *"substituted inline anywhere they appear in skill content, agent content, hook commands, and MCP or LSP server configs."* In dev mode (`--plugin-dir ./`) they resolve to the repo root; in install mode they resolve to the plugin cache.

**Backslashes break under Git Bash even on Windows.** The dbt plugin had 162 backslash-separated path segments (`\scripts\profile_data.py`) that worked in PowerShell but got interpreted as escape characters in Git Bash double-quoted strings. Always use forward slashes inside the path, regardless of OS. *(Source: DBT F7)*

### 6.2 Plugin skill namespace is 2-part `plugin:skill` (and skills appear "locked")

Plugin **skills** are namespaced as 2 parts (`plugin-name:skill-name`), unlike plugin **agents** in a subdirectory layout which are 3 parts. The asymmetry comes from depth: skills are flat single-directory units (`skills/<name>/SKILL.md`), agents are nested (`agents/<name>/agent.md`). *(Source: DBT F8)*

The UI shows plugin skills as **"locked by plugin"** because they live in the plugin cache, not in the user's `~/.claude/skills/`. This means:

- Users **cannot edit them in place** — the plugin cache is managed by Claude Code; local edits get overwritten on `claude plugin update`.
- They **cannot be overridden** by a same-name personal or project skill — they coexist in separate namespaces.
- The canonical customization path is **"fork the plugin"**, not "patch the cached copy."

The practical consequence: **every plugin release is a production deploy**. Users have no escape hatch to fix a shipped bug locally. Test on a fresh install before shipping.

### 6.3 `userConfig` env vars come through subprocesses as `CLAUDE_PLUGIN_OPTION_<KEY>`

> *"All values are exported to plugin subprocesses as `CLAUDE_PLUGIN_OPTION_<KEY>` environment variables. Sensitive values are stored in the system keychain."*
> — [Plugins reference](https://code.claude.com/docs/en/plugins-reference) *(Corroborates DBT F5)*

MCP server blocks remap explicitly in `plugin.json` (`"SQL_SERVER": "${user_config.sql_server}"`), but Python skill scripts invoked via Bash inherit only the prefixed names. Every skill script that reads env vars needs a small helper that runs **at module load time** — before `argparse` evaluates its defaults:

```python
import os

_CONFIG_KEYS = (
    'SQL_SERVER', 'SQL_DATABASE', 'SQL_AUTH_TYPE', 'SQL_USER', 'SQL_PASSWORD',
    'AZURE_TENANT_ID', 'AZURE_CLIENT_ID', 'AZURE_CLIENT_SECRET',
)

def _load_plugin_userconfig_env():
    for key in _CONFIG_KEYS:
        if not os.environ.get(key):
            fallback = os.environ.get(f'CLAUDE_PLUGIN_OPTION_{key}')
            if fallback:
                os.environ[key] = fallback

_load_plugin_userconfig_env()  # MUST run at module load, not inside a lazy .connect() method
```

The dbt plugin's `connect.py` is the reference implementation. Place the helper in every script that reads env vars — placing it only in a shared `connect.py` won't fix argparse defaults in scripts that import `connect.py` lazily inside a method.

### 6.4 `userConfig` may not prompt the user on install

Even with a well-formed `userConfig` block in `plugin.json`, Claude Code sometimes **doesn't prompt** the user for values on install or update. Issue [#39455](https://github.com/anthropics/claude-code/issues/39455) confirms this as an open bug. *(Source: DBT F5 Problem B)*

Defensive design:

1. **Ship a SessionStart hook** that detects missing config and emits a "Setup Required" message at session start. The fabric plugin's `session-start-config-check.py` is the reference pattern.
2. **Document manual `settings.json` paths in the README** as fallback — list the exact key path users edit by hand (`pluginConfigs.<plugin-name>.options.<key>`).
3. **Add a companion `sql-connection` / `fabric-connection` skill** that runs an interactive `configure.py` to write the config explicitly.

### 6.5 Resolve bundled resources via script ancestor walk, not env var alone

`CLAUDE_PLUGIN_ROOT` is a plugin-context variable and is **NOT guaranteed to propagate** to subprocesses launched via Bash from an orchestrator. Issues [#27145](https://github.com/anthropics/claude-code/issues/27145), [#36585](https://github.com/anthropics/claude-code/issues/36585), and [#38699](https://github.com/anthropics/claude-code/issues/38699) confirm it is inconsistent between hooks/skills/agent contexts and not always set in SessionStart and UserPromptSubmit hooks. *(Source: Fabric N16)*

The right resolver builds an **ordered candidate list** and validates with a **sentinel file**:

```python
import os
from pathlib import Path

def resolve_reference_source():
    sentinel = 'm-conversion-risk-catalog.md'  # known file that MUST exist

    candidates = []
    env_root = os.environ.get('CLAUDE_PLUGIN_ROOT')
    if env_root:
        candidates.append(Path(env_root) / 'reference')
    for ancestor in Path(__file__).resolve().parents:
        candidates.append(ancestor / 'reference')

    for c in candidates:
        if (c / sentinel).is_file():
            return c
    return None  # loud failure with remediation note — NOT a stub fallback
```

The script's own `__file__` is always correct regardless of how the subprocess was invoked. Validating by sentinel rejects half-populated stub folders. **Never** ship a "graceful degradation" branch that writes plausible-looking placeholder content; downstream hard-checks will accept the stubs and the real failure surfaces hours later. Loud failure with a remediation note beats silent stubs every time.

### 6.6 Read-only vs mutate-state skill split

Pair-skill design pattern, used by both shipped plugins: one read-only `*-reader` skill and one mutating `*-executor` skill.

| Read-only skill | Mutating sibling | Connection helper |
|---|---|---|
| `sql-server-reader` | `sql-executor` | `sql-connection` |
| `fabric-lakehouse-reader` | `fabric-notebook-deployer` | (Azure auth helper) |

Why split:

- **Reviewers can grant narrower tool access** to the reader (`SELECT` only) than the executor (`INSERT`/`UPDATE`/`DELETE`/`MERGE`).
- **The same script-shape is used for both validation (read) and mutation (write)** — separating them at the skill boundary keeps each SKILL.md focused.
- **Both can share a `*-connection` helper module** — keep connection logic in one place to avoid drift.

Sam Witteveen's *"Agent Skills: Code Beats Markdown"* sharpens this further: there's a third category beyond his "judgment skill vs transformation skill" split — the read-only adapter skill, which we use heavily in the SQL/Fabric pair.

### 6.7 Atomic Bash commands inside SKILL.md examples

Every Bash code block in SKILL.md — example, snippet, usage demo — must be a single atomic operation. **No** `&&`, `||`, `;`, `|`, `$(...)`, backticks, heredocs, subshells. Bulk operations belong in a Python script invoked atomically. *(Source: DBT F9 + Fabric inherited F9)*

The original `fabric-cli-runner/SKILL.md` had a `for` loop with `$(basename ...)` showing bulk notebook import. The pre-shipment audit (`gate_atomic_bash`) caught it; the fix was to replace the loop with a pointer to a new dedicated skill (`fabric-notebook-deployer`) that wraps the bulk operation as a single atomic Python invocation.

Why it matters for skills specifically:

- **Background subagents stall on compound expressions.** Claude Code's permission layer matches rules per-subcommand; compound expressions fall through to interactive prompts that background subagents cannot answer.
- **`acceptEdits` auto-approves atomic filesystem commands** (`mkdir`, `touch`, `mv`, `cp`, `rm`) — but only when atomic.
- **Plugin-level Bash auto-approval hooks** match patterns against individual atomic commands; compound expressions require a fragile splitter.

Rewrite patterns:

- "run A then B" → two separate code blocks
- "pipe A to B" → one Python script doing both, called atomically
- "multiple related commands" → write a wrapper script in `scripts/` and document a single call

### 6.8 PowerShell scripts must be UTF-8 with BOM (or pure ASCII)

If your skill ships or generates PowerShell scripts (`.ps1`), the encoding matters. **PowerShell 5.1** — the only PS edition that handles corporate TLS interception cleanly — falls back to Windows-1252 when no BOM is present. A UTF-8 em-dash (`—`, U+2014, bytes `0xE2 0x80 0x94`) becomes three Windows-1252 characters: `â`, `€`, and `”` (a right double quote). The spurious `”` inside a double-quoted string closes it early, and PS 5.1 then parses the remainder as code, manifesting as cascading "missing terminator" errors many lines away from the real problem. *(Source: Fabric N13)*

Same hazard applies to en-dashes, smart quotes, ellipses, arrows — anything outside 7-bit ASCII.

Mitigations:

- **Python writers** that generate `.ps1`: `open(path, "w", encoding="utf-8-sig")` (the `-sig` variant emits the BOM).
- **Static `.ps1` files** in `scripts/`/`templates/`: UTF-8-with-BOM (`0xEF 0xBB 0xBF` prefix) or strictly 7-bit ASCII. Prefer ASCII when possible.
- **Don't trust pwsh 7's parse-check.** `pwsh.exe` reads UTF-8 by default and does not reproduce the PS 5.1 hazard. CI must parse with the same PowerShell edition the user will run.

### 6.9 Corporate TLS interception breaks every Python-based CLI

If your skill wraps a Python-based CLI (`az`, `fab`, `ms-fabric-cli`, anything using `requests` or `certifi`), it will fail in corporate environments running Norton / Zscaler / Palo Alto / Sophos / NetSkope. The interceptor re-signs HTTPS with its own root CA. Windows trusts it; Python's `certifi` does not — the handshake fails with SSL errors. *(Source: Fabric N12)*

Skills depending on Python CLIs must do at least one of:

1. **Document the `REQUESTS_CA_BUNDLE` workaround prominently** in SKILL.md, BEFORE the user hits the failure. Extract the corporate root CA from the Windows store, append it to a copy of `certifi`'s `cacert.pem`, set `REQUESTS_CA_BUNDLE` to the augmented file.
2. **Prefer Windows-native auth paths** when possible — PowerShell 5.1 + `Connect-*` cmdlets, ODBC Driver 18, .NET HttpClient all trust whatever Windows trusts.
3. **Ship a setup helper script** that runs once per user to configure the augmented cert bundle.

`az login` is listed in countless Microsoft tutorials as the universal auth on-ramp, but in corporate-network environments it's broken by default. Your skill's troubleshooting section should call this out as the **first** thing to check on SSL errors.

### 6.10 CLI flag verification — don't ship phantom flags

When SKILL.md references a flag like `--target` or `--fail-below` in a shell example, that flag **must exist in the script**. LLMs (and humans mid-refactor) will happily invent plausible-sounding flag names. The dbt plugin shipped a SKILL.md documenting `--fail-below 80` for a script that only has `--target 80`; nobody caught it until weeks later. *(Source: DBT F9 round 2.5)*

Mitigation: a pre-commit / CI check that greps every SKILL.md for flags referenced in `python ...` commands, then greps the corresponding script source for those flag names. Mismatches fail loudly.

**Rule:** when documenting a CLI flag in SKILL.md, read the script first. Cost: seconds. Cost of shipping a phantom flag: debugging hours and user confusion.

### 6.11 Skill scripts must respect the background-subagent permission ceiling

`acceptEdits` mode is narrower than its name suggests: it auto-approves file edits and **filesystem Bash** (`mkdir`, `touch`, `mv`, `cp`, `rm`) but **NOT** arbitrary `python` or `dbt` invocations. Background subagents have no interactive channel — every Python-script call inside a background subagent stalls silently waiting for a permission prompt that will never come. *(Source: DBT F9; Changelog v2.1.133 explicitly fixed `"subagents not discovering skills"`)*

The fix is a **plugin-level `PreToolUse` hook** registered in `plugin.json` that matches `Bash` and auto-approves a narrow allowlist of plugin-internal command patterns. The allowlist must recognize the exact path pattern your skill scripts use: `${CLAUDE_PLUGIN_ROOT}/skills/<name>/scripts/<file>.py`. Inline forms like `python -c '<code>'` bypass the script-file pattern and get rejected.

For skill authors:

- **Call other skills' scripts via the documented path pattern.** Never `python -c` inline code that depends on plugin permissions.
- **Keep all heavy lifting in script files**, not in `!`<command>`` injection blocks inside SKILL.md.
- **Document which Bash commands your skill needs** so the plugin author can add them to the hook allowlist.

See `claude-code-plugin-best-practices.md` for the plugin-level hook authoring details.

### 6.12 The "share with author" privacy-preserving skill pattern

When a skill collects content that might help the plugin author improve future releases — unknown M patterns, unrecognized SQL constructs — ship an **opt-in reporter skill** with strong privacy guarantees. The fabric plugin's `report-unknown-patterns` is the reference. *(Source: Fabric `report-unknown-patterns`)*

Requirements:

1. **Explicit per-item preview before any external call.** User sees exactly what will be sent, item by item.
2. **Automatic redaction** of: connection strings, GUIDs, file paths, sheet/table/schema names, customer-identifying strings.
3. **`userConfig` setting with "never / ask / always"** controlling default behavior.
4. **No background or batch reporting.** Each report is a single explicit user action.
5. **The opt-in question is asked exactly once.** Persist the decision.

Default posture: "show me what you're about to send, and let me edit or skip it before it leaves my machine."

---

## 7. Skill Composition with Agents, Hooks, and MCP

Skills are not islands. The canonical 2026 production architecture — codified by Anthropic in its [Agents for Financial Services](https://www.anthropic.com/news/finance-agents) reference templates — bundles three things together:

> *"Each agent template is a reference architecture that packages three things: skills (instructions and domain knowledge for the task), connectors (governed access to the data the task runs on), and subagents."*
> — Agents for Financial Services, 2026-05-05

> *"Skills stack together. Claude automatically identifies which skills are needed and coordinates their use."*
> — [Introducing Agent Skills](https://www.anthropic.com/news/skills)

### 7.1 Agents pre-loading skills via `skills:` frontmatter

A subagent can declare `skills:` in its frontmatter; those skills' **full content** is injected into the agent's system prompt at startup. This is how a "researcher" agent always has `data-profiler` available, or how an "implementer" always has `dbt-runner` loaded:

```yaml
---
name: dbt-implementer
description: Builds staging/mart models. Use after planner approves a plan.
skills:
  - dbt-runner
  - dbt-test-coverage-analyzer
tools: Read, Edit, Bash, Agent(researcher)
---
```

Changelog **v2.1.133** explicitly fixed *"subagents not discovering skills"* — before that fix, subagents silently ignored their declared skills. Always test on a recent Claude Code build.

### 7.2 Skills calling MCP tools

Skill scripts can shell out to MCP tools indirectly via the agent's tool surface. The cleaner pattern: keep MCP as the **always-on connector** for stateful or networked work, and let the skill **orchestrate the call** through markdown instructions.

> *"Skills are pulled on demand, MCP servers are always-on — choose based on call frequency. The team kept the MCP surface tight on purpose; bloat causes context dilution."*
> — Building Claude Code with Boris Cherny (Pragmatic Engineer interview, 2026)

For the local-SQL-server / Fabric-lakehouse case in our own repo, this means: `sql-server-reader` (skill) defines the *workflow* and *redaction rules*, while `local-sql-server-mcp` (MCP) holds the connection.

### 7.3 Skills triggered by hooks

Hooks can call a skill's scripts (via the `command` handler type) to enforce policy at lifecycle events:

- A `SessionStart` hook that runs a skill's `configure.py` to detect missing `userConfig` and emit a "Setup Required" message.
- A `PreToolUse` hook that validates dbt model structure via a skill's analyzer script.
- A `Stop` hook that calls a skill to refresh CLAUDE.md or write a daily-notes summary.

Hooks are *deterministic*; skills are *advisory*. When you absolutely need an action to happen, wrap a skill's script in a hook. When you want Claude to *choose* whether to invoke logic, package it as a skill.

> *"Hooks run scripts automatically at specific points in Claude's workflow. Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens."*
> — [Claude Code best practices](https://code.claude.com/docs/en/best-practices)

### 7.4 Skills + subagents: the `context: fork` pattern

The skill itself can isolate via `context: fork` and `agent: Explore` — running the skill body as a forked subagent task. Useful for research-heavy skills that would otherwise produce >10K tokens of intermediate exploration noise. See section 5's `context: fork` block.

### 7.5 Skills inside plugins

Plugins are the distribution unit. A plugin bundles agents + skills + hooks + MCP server configs into one installable. *(Validated by `AICodeKing` April 2026 roundup: "Plugins are distribution units bundling skills + hooks + sub-agents + MCP servers — installing one gets the whole bundle, which is what differentiates plugins from raw skills.")*

> *"Plugins are a lightweight way to package and share any combination of: Slash commands, Subagents, MCP servers, and Hooks."*
> — [Customize Claude Code with plugins](https://www.anthropic.com/news/claude-code-plugins)

The official Anthropic-managed plugin directory ([github.com/anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)) launched **2026-05-22**, consolidating "30-plus internal and 15 vetted external Claude Code extensions behind a two-tier review system."

See `claude-code-plugin-best-practices.md` for plugin authoring patterns.

---

## 8. Anti-Patterns

| Anti-pattern | Why it fails |
|---|---|
| **Vague description** (`Helps with X`) | Loader can't route; skill never triggers. Description must lead with trigger verbs. |
| **Putting scripts in agent.md when they belong in a skill** | Mixes orchestration with execution; harder to reuse across agents. |
| **Bare `python <skill>/scripts/foo.py` paths** | Resolves wrong in plugin cache; use `${CLAUDE_SKILL_DIR}` / `${CLAUDE_PLUGIN_ROOT}`. |
| **Assuming `CLAUDE_PLUGIN_ROOT` propagates to subprocesses** | It doesn't reliably. Walk `__file__` ancestors + validate by sentinel. |
| **Bare `<KEY>` env var reads inside plugin skills** | Plugin userConfig comes through as `CLAUDE_PLUGIN_OPTION_<KEY>`. Add the helper at module load. |
| **Compound bash in SKILL.md** (`&&`, `||`, `;`, `|`, `$(...)`) | Background subagents stall; permission layer can't auto-approve. |
| **Embedding phantom CLI flags** | Document only flags that exist in the script. CI-grep the cross-reference. |
| **Hardcoded secrets in scripts** | Read from env vars; fail loudly when missing. |
| **Massive prose-heavy SKILL.md (>800 lines)** | Push detail into `references/` and link. Use headings, not prose. |
| **Silent fallback / stub generation when resources are missing** | Loud failure with a remediation note. Downstream hard-checks will accept stubs and the real failure surfaces hours later. |
| **`python -c` inline code from skill examples** | Plugin allowlist won't match it. Always use script files. |
| **Hardcoded `$HOME/.claude/skills/` paths** | Wrong for plugin-installed skills. Use the substitution variables. |
| **UTF-8 em-dashes in PS5.1 `.ps1` files without BOM** | Cascading "missing terminator" errors. Use ASCII or UTF-8-sig. |
| **Skill triggered "always on" (no `description` filtering)** | Bloats context for every session. That's what MCP is for. |
| **One skill doing both read + mutate** | Reviewers can't grant narrow access. Split into `*-reader` + `*-executor` + shared `*-connection`. |

---

## 9. External Source Validation

| Source | URL | Load-bearing quote |
|---|---|---|
| **Introducing Agent Skills** (blog, 2025-10-16) | [anthropic.com/news/skills](https://www.anthropic.com/news/skills) | *"Skills are folders that include instructions, scripts, and resources that Claude can load when needed."* + *"Skills stack together. Claude automatically identifies which skills are needed and coordinates their use."* |
| **Equipping agents for the real world with Agent Skills** (engineering, 2025-10-16) | [anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) | *"A skill is a directory that contains a `SKILL.md` file."* + *"Progressive disclosure is the core design principle that makes Agent Skills flexible and scalable."* |
| **Agents for Financial Services** (blog, 2026-05-05) | [anthropic.com/news/finance-agents](https://www.anthropic.com/news/finance-agents) | *"Each agent template is a reference architecture that packages three things: skills... connectors... and subagents."* — canonical "skills + connectors + subagents" trio. |
| **Claude Code Skills doc** | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) | Authoritative frontmatter and behavior spec. |
| **Plugins reference** | [code.claude.com/docs/en/plugins-reference](https://code.claude.com/docs/en/plugins-reference) | *"All values are exported to plugin subprocesses as `CLAUDE_PLUGIN_OPTION_<KEY>` environment variables."* |
| **Best practices for Claude Code** (live doc) | [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices) | *"For domain knowledge or workflows that are only relevant sometimes, use skills instead. Claude loads them on demand without bloating every conversation."* |
| **Skills explained** (blog) | [claude.com/blog/skills-explained](https://claude.com/blog/skills-explained) | Decision matrix for skills vs subagents vs MCP vs prompts. |
| **Changelog v2.1.142** | [github.com/anthropics/claude-code](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) | *"Plugins with root-level SKILL.md and no skills/ subdirectory now surfaced as skill."* |
| **Changelog v2.1.145** | same | *"Fixed `claude plugin validate` not flagging `skills:` entries pointing at files instead of directories."* |
| **Changelog v2.1.133** | same | *"Fixed subagents not discovering skills."* |
| **Changelog v2.1.126** | same | *"`claude_code.skill_activated` event with `invocation_trigger`"* — telemetry for skill invocations. |
| **Changelog v2.1.121** | same | *"Added filter search to `/skills`."* |
| **Changelog v2.1.120** | same | *"Skills reference effort with `${CLAUDE_EFFORT}`."* |
| **Changelog v2.1.119** | same | *"Fixed pre-compaction skill re-execution."* |
| **Changelog v2.1.111** | same | *"Added `/less-permission-prompts` skill"* + *"Fixed Claude calling non-existent skill."* |
| **Anthropic skills repo** | [github.com/anthropics/skills](https://github.com/anthropics/skills) | Reference layouts for docx/pdf/pptx/xlsx, frontend-design, skill-creator, mcp-builder. |
| **Open standard** | [agentskills.io](https://agentskills.io) | Skills as cross-platform open standard (2025-12-18). |
| **Sam Witteveen — Code Beats Markdown** | [youtube.com/watch?v=IjiaCOt7bP8](https://www.youtube.com/watch?v=IjiaCOt7bP8) | Markdown is for *judgment* tasks; code is for *transformation* tasks. |
| **Anthropic's Full Claude Skills Guide In 22 Minutes** | [youtube.com/watch?v=TzJecWCbex0](https://www.youtube.com/watch?v=TzJecWCbex0) | Description-as-routing-signal, anthropics/skills as canonical reference. |
| **Building Claude Code with Boris Cherny** (Pragmatic Engineer) | [newsletter.pragmaticengineer.com/p/building-claude-code-with-boris-cherny](https://newsletter.pragmaticengineer.com/p/building-claude-code-with-boris-cherny) | "Skills are pulled on demand, MCP servers are always-on — choose by call frequency. Plug-in authors should default to skills first." |

For plugin-shipped skill issues, see open GitHub issues:
- [#27145](https://github.com/anthropics/claude-code/issues/27145) — `CLAUDE_PLUGIN_ROOT` not set for SessionStart hooks
- [#36585](https://github.com/anthropics/claude-code/issues/36585) — `CLAUDE_PLUGIN_ROOT` not passed to UserPromptSubmit hooks
- [#38699](https://github.com/anthropics/claude-code/issues/38699) — `CLAUDE_PLUGIN_ROOT` inconsistent between hooks/skills/agent
- [#39455](https://github.com/anthropics/claude-code/issues/39455) — Plugin userConfig values not prompted on enable

---

## 10. Distribution & Sharing

| Scope | Location | Sharing |
|---|---|---|
| **Personal** | `~/.claude/skills/<name>/` | Not shared by default |
| **Project** | `<project>/.claude/skills/<name>/` | Via git |
| **Plugin** | bundled inside plugin's `skills/` dir | `plugin-name:skill-name` namespace, distributed via marketplace |
| **Managed / Enterprise** | deployed org-wide via managed settings | Takes precedence over user/project skills |

**Name conflict priority:** Enterprise > Personal > Project.

**Slash commands = skills** (late 2025 merger). Both create `/deploy`:
- `.claude/commands/deploy.md`
- `.claude/skills/deploy/SKILL.md`

Skill form is preferred because it supports supporting files, richer frontmatter, and automatic loading.

### Bundled Claude Code skills

| Skill | What it does |
|-------|--------------|
| `/batch <instructions>` | Decomposes a task into 5-30 units, spawns one background agent per unit in isolated worktrees, each opens a PR |
| `/simplify [focus]` | Spawns 3 review agents in parallel, aggregates findings, applies simplifications |
| `/loop [interval] <prompt>` | Runs a prompt on a cron schedule (default 10m) |
| `/debug [description]` | Enables debug logging mid-session and reads the logs |
| `/claude-api` | Loads Claude API reference for the language it detects |
| `/less-permission-prompts` | Anthropic-shipped (v2.1.111) permission-trimming skill |

### Official Anthropic community skills

Deployed from [`anthropics/skills`](https://github.com/anthropics/skills) (often vendored as a git submodule): `docx`, `pdf`, `pptx`, `xlsx`, `frontend-design`, `skill-creator`, `mcp-builder`, `theme-factory`, `web-artifacts-builder`, `webapp-testing`.

---

## 11. Pre-Release Checklist

- [ ] `SKILL.md` at root of skill folder
- [ ] YAML frontmatter present and valid
- [ ] `name` follows naming rules (lowercase, hyphens, ≤64 chars)
- [ ] `description` is specific and includes triggers (≤1024 chars)
- [ ] Markdown body is <500 lines (ideally <200)
- [ ] All Bash code blocks are **atomic** (no `&&`, `||`, `;`, `|`, `$(...)`)
- [ ] All script paths use `${CLAUDE_SKILL_DIR}` or `${CLAUDE_PLUGIN_ROOT}` with forward slashes
- [ ] Every CLI flag documented in SKILL.md exists in the corresponding script
- [ ] Scripts read env vars via the `CLAUDE_PLUGIN_OPTION_<KEY>` fallback helper
- [ ] Resource resolver walks script ancestors + validates with a sentinel
- [ ] No hardcoded secrets; no hardcoded local paths (`C:\Users\...`, `/home/...`)
- [ ] No phantom flags; CI greps for SKILL.md flags vs script source
- [ ] PowerShell scripts saved as UTF-8-with-BOM or strict ASCII
- [ ] Reader/executor split if the skill mutates state
- [ ] Tested on a **fresh install** (plugin cache, not dev mode), with `userConfig` empty
- [ ] Background-subagent permission ceiling respected (allowlist patterns for all script paths)
- [ ] Reference files linked from SKILL.md; no broken links

---

## 12. Further Reading

### Companion docs in this handbook

- [`how-to-create-skills.md`](how-to-create-skills.md) — step-by-step build guide
- [`claude-code-agent-best-practices.md`](claude-code-agent-best-practices.md) — agent design patterns
- [`claude-code-hooks-best-practices.md`](claude-code-hooks-best-practices.md) — hook design patterns
- [`claude-code-plugin-best-practices.md`](claude-code-plugin-best-practices.md) — plugin-author perspective on skill-in-plugin issues

### Top-3 skill-relevant videos

1. **Anthropic's Full Claude Skills Guide In 22 Minutes** — [youtube.com/watch?v=TzJecWCbex0](https://www.youtube.com/watch?v=TzJecWCbex0) — compressed authoritative tour; cite for "description is a routing signal, not documentation."
2. **Agent Skills: Code Beats Markdown — Sam Witteveen** — [youtube.com/watch?v=IjiaCOt7bP8](https://www.youtube.com/watch?v=IjiaCOt7bP8) — justification for the `scripts/` + `SKILL.md` split (judgment vs transformation).
3. **Claude Code's Creator Uses These Claude Skills Every Single Day** — [youtube.com/watch?v=AhXfI1rSUPc](https://www.youtube.com/watch?v=AhXfI1rSUPc) — what Anthropic's own team runs internally; confirms short structured SKILL.md beats long prose.

### Top-3 skill-relevant blog posts

1. **Introducing Agent Skills (2025-10-16)** — [anthropic.com/news/skills](https://www.anthropic.com/news/skills) — the launch post; the canonical "what are skills" framing.
2. **Equipping agents for the real world with Agent Skills (2025-10-16)** — [anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) — engineering deep-dive on progressive disclosure.
3. **Agents for Financial Services (2026-05-05)** — [anthropic.com/news/finance-agents](https://www.anthropic.com/news/finance-agents) — canonical "skills + connectors + subagents" production template.

### Open standards / repos

- [agentskills.io](https://agentskills.io) — Agent Skills open standard
- [github.com/anthropics/skills](https://github.com/anthropics/skills) — Anthropic's official skill reference
- [github.com/travisvn/awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) — community skill catalog
- [github.com/anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official) — official plugin directory (launched 2026-05-22)
