# How to Create Claude Code Agents

**Last Updated:** 2026-05-26

This guide provides comprehensive instructions for creating custom agents (subagents) in Claude Code.

> **Companion documents:**
> - `claude-code-agent-best-practices.md` — patterns, multi-agent orchestration, modern frontmatter reference (canonical)
> - `how-to-create-skills.md` — step-by-step skill guide
> - `how-to-create-hooks.md` — step-by-step hook guide
> - `how-to-create-plugins.md` — step-by-step plugin guide
> - `claude-code-skill-best-practices.md` — skill-level patterns
> - `claude-code-hooks-best-practices.md` — hook patterns
> - `claude-code-plugin-best-practices.md` — plugin patterns

## Table of Contents

1. [What Are Agents?](#what-are-agents)
2. [Agent File Structure](#agent-file-structure)
3. [YAML Frontmatter Reference](#yaml-frontmatter-reference)
4. [Step-by-Step Creation Process](#step-by-step-creation-process)
5. [Tools Configuration](#tools-configuration)
6. [Model Selection](#model-selection)
7. [Best Practices](#best-practices)
8. [Common Patterns](#common-patterns)
9. [Delegation and Orchestration](#delegation-and-orchestration)
10. [Examples](#examples)

---

## What Are Agents?

**Agents** (also called subagents) are specialized AI assistants that handle specific types of tasks in Claude Code. Each agent:

- Runs in its own isolated context window
- Has a custom system prompt (the agent.md content)
- Can have restricted tool access
- Works independently and returns results
- Can use a different model (Sonnet, Opus, or Haiku)

### Key Benefits

1. **Preserve Context**: Keep exploration and implementation out of your main conversation
2. **Enforce Constraints**: Limit which tools an agent can use (e.g., read-only reviewers)
3. **Reuse Configurations**: User-level agents are available across all projects
4. **Specialize Behavior**: Focused system prompts for specific domains
5. **Control Costs**: Route simple tasks to faster, cheaper models like Haiku

### When to Use Agents vs Task() Tool

**Custom Agents** (agent.md files):
- For frequently reused roles (code reviewer, security auditor, etc.)
- When you need strict tool restrictions
- For domain expertise that spans multiple projects
- When you want automatic delegation based on task descriptions

**Task() Tool** (Master-Clone architecture):
- For one-off delegations or project-specific orchestration
- When all context is already in claude.md
- For flexible delegation without maintaining separate agent files
- When you prefer dynamic rather than static specialization

> **⚠ Master-Clone runs from the main thread only.** Claude Code enforces a one-level subagent hierarchy — **subagents cannot spawn subagents**. The Task() patterns shown later in this document only work when invoked from the main conversation, not from inside a subagent prompt. For multi-stage workflows from a subagent context, use parent-driven loops, Agent Teams, or external orchestration. See `claude-code-agent-best-practices.md` § 1.1 and § 6.4 for the full constraint.

**This Project Uses**: Master-Clone architecture with Task() tool for flexibility. Custom agents are created when:
- The role is reused across multiple sessions
- Tool restrictions are critical for safety
- The expertise requires extensive system prompt context

---

## Agent File Structure

A Claude Code agent is a Markdown file with YAML frontmatter:

```
agent-name/
└── agent.md          # Required: YAML frontmatter + markdown instructions
```

### File Naming

The file **must** be named `agent.md` and placed in a folder with the agent's name.

### Directory Structure

```
~/.claude/agents/              # User-level (all projects)
  ├── code-reviewer/
  │   └── agent.md
  └── security-auditor/
      └── agent.md

<project-root>/.claude/agents/ # Project-level (this project only)
  ├── dbt-developer/
  │   └── agent.md
  └── semantic-modeler/
      └── agent.md
```

**For this project**, all agents are created in:
```
Agents/<agent-name>/agent.md
```

Then manually copied to `~/.claude/agents/<agent-name>/` (user-level) or `<project-root>/.claude/agents/<agent-name>/` (project-level) as needed. See the deployment block in the project root `CLAUDE.md`.

---

## YAML Frontmatter Reference

### Required Fields

```yaml
---
name: agent-name
description: >
  Clear description of when to delegate to this agent.
  Triggers automatic delegation based on task matching.
---
```

### Optional Fields

```yaml
---
name: agent-name
description: >
  Clear description of when to delegate to this agent.
tools: Read, Grep, Glob, Bash
disallowedTools: Write, Edit
model: haiku
permissionMode: default
skills: skill-name-1, skill-name-2
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate.sh"
---
```

### Complete Field Reference Table

> **Canonical reference:** `claude-code-agent-best-practices.md` § 3 lists all 14 modern fields with their semantics, defaults, and the **plugin-context stripping rules** (some fields are silently dropped for plugin-shipped agents). The table below is a quick lookup — consult § 3 of the best-practices doc for full detail.

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | Yes | String | Unique identifier (lowercase, hyphens only) |
| `description` | Yes | String | When Claude should delegate to this agent |
| `tools` | No | String/List | Tools the agent can use (inherits all if omitted) |
| `disallowedTools` | No | String/List | Tools to explicitly deny |
| `model` | No | String | Model: `sonnet`, `opus`, `haiku`, or `inherit` (default: `inherit`) |
| `permissionMode` | No | String | Permission behavior: `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, `plan` *(stripped in plugin context)* |
| `maxTurns` | No | Integer | Cap on agentic turns before forced stop |
| `skills` | No | String/List | Skills to load into agent context (2-part `plugin:skill` namespace in plugin context) |
| `mcpServers` | No | List | MCP servers scoped to this agent *(stripped in plugin context)* |
| `hooks` | No | Object | Lifecycle hooks (PreToolUse, PostToolUse, SubagentStart, SubagentStop) *(stripped in plugin context)* |
| `memory` | No | String | Persistent memory dir: `user` / `project` / `local` |
| `background` | No | Boolean | Run as background task by default |
| `effort` | No | String | Reasoning effort: `low` / `medium` / `high` / `max` / `xhigh` (Opus 4.7+) |
| `isolation` | No | String | Only `worktree` — runs the agent in a temp git worktree |
| `color` | No | String | UI color: red / blue / green / yellow / purple / orange / pink / cyan |
| `initialPrompt` | No | String | Auto-submitted first turn when run as a `--agent` main session |

### Field Specifications

#### `name` (Required)

- **Type**: String
- **Validation**:
  - Maximum 64 characters
  - Lowercase letters, numbers, and hyphens only
  - No XML tags
  - No reserved words
- **Example**: `dbt-developer`, `code-reviewer`, `security-auditor`

#### `description` (Required)

- **Type**: String (can use multi-line YAML `>` notation)
- **Validation**:
  - Maximum 1024 characters
  - Non-empty
  - No XML tags
- **Purpose**: Triggers automatic delegation. Claude matches tasks to agents based on this description.
- **Best Practice**:
  - Start with when to use the agent
  - Describe what it does proactively
  - Mention key capabilities
  - Include trigger keywords

**Example**:
```yaml
description: >
  Expert code reviewer for data engineering and BI projects.
  Proactively reviews Python, SQL, and DAX code for security,
  performance, and standards compliance. Use when code changes
  need quality assurance before deployment.
```

#### `tools` (Optional)

- **Type**: Comma-separated string or YAML list
- **Default**: Inherits all tools from parent context (including MCP tools)
- **Purpose**: Restrict which Claude Code built-in tools the agent can use
- **Format Options**:
  ```yaml
  # Option 1: Comma-separated string
  tools: Read, Grep, Glob

  # Option 2: YAML list
  tools:
    - Read
    - Grep
    - Glob
  ```

**Available Tools**:
- `Read` - Read files
- `Write` - Create new files
- `Edit` - Modify existing files
- `Glob` - Find files by pattern
- `Grep` - Search file contents
- `Bash` - Execute shell commands
- `WebFetch` - Fetch web content
- `WebSearch` - Search the web
- `Task` - Spawn additional subagents (this is available for future use but currently does not work subagents cannot call other subagents only the main chat agent can use this tool.)
- And more (see Claude Code docs)

**Tool Access Patterns** — see the canonical "Tool Access by Role" reference in the [Tools Configuration](#tools-configuration) section below for the full pattern catalog (read-only, research, developer, orchestrator, security auditor).

#### `model` (Optional)

- **Type**: String
- **Values**: `sonnet`, `opus`, `haiku`
- **Default**: Inherits from parent context
- **Purpose**: Use different model for cost/performance tradeoffs

**When to Use Each Model**:
```yaml
# Haiku - Fast and cheap
model: haiku
# Use for: Code formatting, simple reviews, documentation generation

# Sonnet - Balanced (default)
model: sonnet
# Use for: Most development tasks, moderate complexity

# Opus - Most capable (expensive)
model: opus
# Use for: Complex architecture, critical security reviews
```

#### `disallowedTools` (Optional)

- **Type**: Comma-separated string or YAML list
- **Default**: No tools disallowed
- **Purpose**: Explicitly deny specific tools (alternative to allowlist with `tools`)
- **Use Case**: When you want agent to inherit most tools but block specific ones

**Example**:
```yaml
# Allow all tools except Write and Edit (read-only agent)
disallowedTools: Write, Edit
```

#### `permissionMode` (Optional)

- **Type**: String
- **Values**: `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, `plan`
- **Default**: `default`
- **Purpose**: Control how the agent handles permission prompts

**Permission Modes**:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `default` | Standard permission checking with prompts | Normal interactive agents |
| `acceptEdits` | Auto-accept file edits | Trusted code generators |
| `auto` | Auto-approve harness-allowlisted tools | Headless / scripted runs |
| `dontAsk` | Auto-deny unpermitted operations | Background agents |
| `bypassPermissions` | Skip all permission checks (dangerous!) | Fully trusted automation |
| `plan` | Read-only exploration mode | Research and planning agents |

**Example**:
```yaml
# Auto-accept file edits for trusted code generator
permissionMode: acceptEdits
```

#### `skills` (Optional)

- **Type**: Comma-separated string or YAML list
- **Default**: No skills loaded
- **Purpose**: Load specific skills into agent context

> **⚠ Standalone vs plugin context.** Bare skill names (e.g. `skills: dbt-runner`) only resolve when the agent lives under `~/.claude/agents/` or `<project>/.claude/agents/` alongside a matching skill under `~/.claude/skills/`. For agents **shipped inside a plugin**, the bare form silently produces an empty preload — you must use the 2-part `plugin:skill` namespace (e.g. `skills: dbt-pipeline-toolkit:dbt-runner`). See `claude-code-agent-best-practices.md` § 6.14 for the full failure mode.

- **Format**:
  ```yaml
  # Standalone agent — bare name
  skills: dbt-runner

  # Plugin-shipped agent — 2-part namespace required
  skills: dbt-pipeline-toolkit:dbt-runner

  # Multiple skills (comma-separated)
  skills: dbt-runner, sql-formatter

  # Multiple skills (YAML list)
  skills:
    - dbt-runner
    - sql-formatter
  ```

**Example**:
```yaml
# Load dbt-runner skill for analytics engineering agent (standalone)
skills: dbt-runner
```

#### `hooks` (Optional)

- **Type**: YAML object
- **Purpose**: Define lifecycle hooks that run during agent execution
- **Available Hooks**:
  - `PreToolUse`: Before tool execution
  - `PostToolUse`: After tool execution
  - `SubagentStart`: When subagent starts
  - `SubagentStop`: When subagent stops

**Example**:
```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh $TOOL_INPUT"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
```

**Hook Structure**:
- `matcher`: Regex pattern matching tool name
- `hooks`: Array of hook definitions
  - `type`: `command` (run shell command)
  - `command`: Command to execute (can use environment variables)

---

## Agent Scope and Priority

Agents can be stored in different locations with different priority levels:

| Location | Scope | Priority | Use Case |
|----------|-------|----------|----------|
| `--agents` CLI flag | Current session only | 1 (highest) | Quick testing |
| `.claude/agents/` | Current project | 2 | Team collaboration (check into git) |
| `~/.claude/agents/` | All your projects | 3 | Personal reuse |
| Plugin's `agents/` directory | Where plugin enabled | 4 (lowest) | Distribution |

**Priority Rules**:
- If multiple agents have same name, higher priority wins
- Project-level agents (`.claude/agents/`) override user-level (`~/.claude/agents/`)
- CLI-defined agents override all file-based agents

---

## Built-in Subagents

Claude Code includes pre-configured subagents available out-of-the-box:

1. **Explore** - Fast, read-only codebase analysis using Haiku model
2. **Plan** - Research agent for plan mode (read-only exploration)
3. **General-purpose** - Complex multi-step tasks with full tool access
4. **Bash** - Terminal command execution specialist
5. **statusline-setup** - Status line configuration agent

You can override built-in agents by creating your own with the same name at higher priority level.

---

## Step-by-Step Creation Process

### Step 1: Plan the Agent's Role

Before writing any code, define:

**Purpose**:
- What specific expertise does this agent provide?
- What types of tasks should trigger delegation?

**Scope**:
- What should this agent do?
- What should it NOT do?

**Tools Needed**:
- Does it need to write code? (Write, Edit)
- Does it need to execute commands? (Bash)
- Should it be read-only? (Read, Grep, Glob)

**Model Selection**:
- Is this a simple, repetitive task? (Haiku)
- Does it require deep reasoning? (Sonnet/Opus)

### Step 2: Create the Directory Structure

```bash
# For this repo
mkdir -p Agents/agent-name
```

### Step 3: Create agent.md File

Create `agent.md` with YAML frontmatter and instructions:

```markdown
---
name: agent-name
description: >
  Clear description that triggers delegation.
  What does this agent do and when to use it.
tools: Read, Write, Edit, Glob, Grep, Bash
model: sonnet
---

# Agent Name

You are a [role description].

## Your Role

[Clear statement of responsibility]

## Your Expertise

- Expertise area 1
- Expertise area 2
- Expertise area 3

## Available Skills

[List any skills this agent can use]

## Standards and Guidelines

[Project-specific standards, naming conventions, etc.]

## Workflow

[How this agent should approach tasks]

## Communication and Handoffs

[How to receive tasks and deliver results]

## Best Practices Checklist

[Checklist before completing work]

## Success Criteria

[How to measure success]
```

### Step 4: Write the System Prompt

The markdown content becomes the agent's system prompt. Include:

1. **Identity**: "You are a senior X specializing in Y"
2. **Role**: Clear statement of responsibility
3. **Expertise**: List of capabilities
4. **Standards**: Project-specific guidelines
5. **Workflow**: How to approach tasks
6. **Communication**: How to interact with users and other agents
7. **Success Criteria**: What defines completion

### Step 5: Test the Agent

**Using /agents Command**:
```bash
# Open Claude Code
/agents

# Select "create new agent" and test delegation
```

**Manual Testing**:
1. Place agent.md in appropriate directory
2. Start Claude Code session
3. Give a task that matches the description
4. Verify Claude delegates to the agent

**Or Explicitly Invoke**:
```bash
claude-code --agent agent-name "specific task"
```

### Step 6: Iterate and Refine

Based on testing:
- Refine the description for better delegation matching
- Adjust tool access if needed
- Update instructions for clarity
- Add examples and patterns

### Step 7: Deploy

**User-level** (available in all projects):
```bash
cp Agents/agent-name/agent.md ~/.claude/agents/agent-name/
```

**Project-level** (this project only):
```bash
cp Agents/agent-name/agent.md .claude/agents/agent-name/
```

---

## Tools Configuration

### Whitelist Approach (Recommended)

Explicitly list only the tools the agent needs:

```yaml
tools: Read, Grep, Glob  # Read-only agent
```

### Inherit All Tools

Omit the `tools` field to inherit parent's tools (including MCP tools):

```yaml
# No tools field = inherits all available tools
---
name: full-access-agent
description: Agent with full tool access
---
```

### Tool Access by Role

**Code Reviewer** (Read-only):
```yaml
tools: Read, Grep, Glob
```

**Research Agent**:
```yaml
tools: Read, Grep, Glob, WebFetch, WebSearch
```

**Developer Agent**:
```yaml
tools: Read, Write, Edit, Bash, Glob, Grep
```

**Project Manager** (Orchestrator):
```yaml
tools: Read, Grep, Glob, Task
```

**Security Auditor** (Read + Research):
```yaml
tools: Read, Grep, Glob, WebFetch, WebSearch
```

---

## Model Selection

> **Canonical reference:** `claude-code-agent-best-practices.md` § 5 covers the current model lineup (Haiku 4.5, Sonnet 4.5/4.6, Opus 4.5/4.6/4.7), the `effort` parameter (`low` / `medium` / `high` / `max` / `xhigh`), and the cost-vs-capability tradeoffs in detail. The summary below is a quick lookup.

### Cost and Performance Tradeoffs (2026)

| Model family | Latest | Best For |
|---|---|---|
| Haiku | 4.5 (`claude-haiku-4-5-20251001`) | Fast, cheap — formatting, simple reviews, deterministic transforms |
| Sonnet | 4.6 (`claude-sonnet-4-6`) | Balanced default — most coding, multi-file edits, code review |
| Opus | 4.7 (`claude-opus-4-7`) | Deepest reasoning — architecture, security audits, gnarly debugging; `xhigh` effort tier available |

### When to Use Each Model

```yaml
# Haiku - Fast and cheap
model: haiku
# Use for: formatting, lint-style review, doc generation, test execution

# Sonnet - Balanced default
model: sonnet
# Use for: most coding tasks, dbt models, DAX measures, code review

# Opus - Deepest reasoning
model: opus
effort: high   # or xhigh for Opus 4.7 — see best-practices § 5 for caveats
# Use for: architecture, security audits, complex modeling decisions
```

Prefer omitting `model` entirely (default `inherit`) for agents that should follow the parent session's choice. Hard-pin a model only when the agent's quality demands it.

---

## Best Practices

### 1. Clear Descriptions for Delegation

Write descriptions that help Claude match tasks to agents:

**Good**:
```yaml
description: >
  Expert dbt developer for analytics engineering.
  Creates staging, intermediate, and mart models following
  Kimball methodology. Use when building or modifying
  dbt transformations, data models, or tests.
```

**Bad**:
```yaml
description: Helps with dbt
```

**Key Elements**:
- Start with the role/expertise
- List primary responsibilities
- Include trigger keywords (when to use)
- Mention key deliverables

### 2. Principle of Least Privilege

Only grant tools the agent needs:

```yaml
# Good - Code reviewer doesn't need Write
tools: Read, Grep, Glob

# Bad - Unnecessary write access
tools: Read, Write, Edit, Bash
```

### 3. Focused Expertise

Each agent should have a single, clear focus:

**Good** - Separate agents:
- dbt-developer: Data modeling
- semantic-modeler: DAX measures
- pbi-developer: Report visuals

**Bad** - One mega-agent:
- data-platform-developer: Everything from SQL to visuals

### 4. Include Project Context

Reference project standards in the agent prompt:

```markdown
## Standards

Follow the conventions in `claude.md`:
- dbt models: stg_, int_, fct_, dim_ prefixes
- SQL style: lowercase keywords, CTEs, qualified columns
- Testing: unique + not_null on all primary keys
```

### 5. Define Success Criteria

Clear checklist for when work is complete:

```markdown
## Success Criteria

You are successful when:
- ✅ All models compile without errors
- ✅ All tests pass
- ✅ Documentation is complete
- ✅ Code follows style guide
- ✅ Handoff information is clear
```

### 6. Human-in-the-Loop Patterns

For critical operations, agents should:
- Ask for confirmation before destructive operations
- Provide clear summaries of what will be changed
- Offer options when multiple valid approaches exist
- Surface blockers and errors clearly

### 7. Definition of Done

Each agent should have a clear "Definition of Done":

```markdown
## Definition of Done

Before marking work complete:
1. All code compiles and runs successfully
2. All tests pass
3. Documentation is updated
4. Code review checklist completed
5. Handoff notes prepared for next agent
```

---

## Common Patterns

### Pattern 1: Read-Only Reviewer

**Use Case**: Code review, security audit, standards compliance

```yaml
---
name: code-reviewer
description: >
  Expert code reviewer for data and BI projects.
  Reviews SQL, Python, and DAX for security, performance,
  and standards compliance. Use after development before deployment.
tools: Read, Grep, Glob, WebFetch, WebSearch
model: sonnet
---

# Code Reviewer

You are a senior code reviewer specializing in data engineering and BI.

## Your Role

Review code changes for:
- Security vulnerabilities
- Performance issues
- Standards compliance
- Best practices

## Available Tools

- Read: Examine code files
- Grep: Search for patterns
- Glob: Find files
- WebFetch/WebSearch: Research best practices

## Review Process

1. Read all changed files
2. Check against standards
3. Identify issues
4. Provide recommendations
5. Summarize findings

[Additional sections...]
```

### Pattern 2: Code Writer

**Use Case**: Development, implementation, code generation

```yaml
---
name: dbt-developer
description: >
  Expert dbt developer for analytics engineering.
  Creates staging, intermediate, and mart models.
  Use when building data transformations.
tools: Read, Write, Edit, Bash, Glob, Grep
model: sonnet
---

# dbt Developer

You are a senior analytics engineer specializing in dbt and SQL Server.

## Your Role

Build robust data transformation pipelines using dbt.

## Available Skills

- dbt-runner: Execute dbt commands

## Development Workflow

1. Understand requirements
2. Create model files
3. Document in schema.yml
4. Compile and test
5. Provide handoff notes

[Additional sections...]
```

### Pattern 3: Research Agent

**Use Case**: Investigation, documentation, analysis

```yaml
---
name: tech-researcher
description: >
  Research specialist for technical documentation and best practices.
  Searches web, reads docs, synthesizes findings.
  Use when investigating new technologies or patterns.
tools: Read, Grep, Glob, WebFetch, WebSearch
model: haiku
---

# Tech Researcher

You are a technical researcher specializing in data and BI technologies.

## Your Role

Research and synthesize information about:
- New technologies
- Best practices
- Architecture patterns
- Tool capabilities

## Research Process

1. Search relevant documentation
2. Read official sources
3. Identify patterns
4. Synthesize findings
5. Provide recommendations

[Additional sections...]
```

### Pattern 4: Orchestrator

**Use Case**: Project management, multi-agent coordination

```yaml
---
name: project-manager
description: >
  Senior project manager coordinating data platform specialists.
  Breaks down complex requests, delegates to specialists,
  consolidates results. Use for end-to-end solutions.
tools: Read, Grep, Glob, Task
model: sonnet
---

# Project Manager

You are a senior data platform project manager.

## Your Role

Coordinate specialists to deliver complete solutions:
- Break complex requests into phases
- Delegate to appropriate agents using Task()
- Manage handoffs between agents
- Consolidate results

## Available Specialists

- dbt-developer
- semantic-modeler
- pbi-developer
- data-qa
- code-reviewer

[Additional sections...]
```

---

## Delegation and Orchestration

### Automatic Delegation

Claude automatically delegates based on:
1. Task description in user request
2. Agent's `description` field
3. Current context
4. Available tools

**Example**:

User: "Review the SQL code for security issues"

Claude sees:
- code-reviewer agent with description: "Reviews SQL...for security"
- Automatically delegates to code-reviewer

### Explicit Invocation

**Via CLI**:
```bash
claude-code --agent agent-name "specific task"
```

**Via Task() Tool** (in agent prompts):
```python
# From orchestrator agent using Task() tool
Task(
    subagent_type="general-purpose",
    task="Create dbt models using dbt-developer expertise",
    description="Create dbt models"
)
```

### Master-Clone Architecture (Used in This Project)

Instead of rigid specialized agents, use Task() from the main thread to spawn clones:

> **⚠ Main thread only.** Subagents cannot spawn subagents — Claude Code enforces a one-level hierarchy. The Master-Clone pattern below is only valid from the main conversation (or a `--agent` main session). A regular subagent that tries to call Task() will fail. See `claude-code-agent-best-practices.md` § 6.4.

**Main Agent** (claude.md context):
- All project context loaded
- Access to all agents' expertise descriptions

**Task() Spawning** (from main thread):
```python
# Spawn specialist clone for dbt work
Task(
    subagent_type="general-purpose",
    task="Act as dbt-developer: Create sales fact table following standards in claude.md",
    model="sonnet"
)
```

**Benefits**:
- Flexible: Each clone gets full project context
- Dynamic: Can combine multiple specialties in one task
- Simple: No need to maintain separate agent files for one-off tasks

**When to Use Custom Agents Instead**:
- Frequently reused across sessions
- Need strict tool restrictions for safety
- Require extensive domain-specific system prompt
- Called from a subagent context (where Master-Clone won't work)

### Foreground vs Background Execution

Agents can run in two modes:

**Foreground Mode** (Default):
- Blocks main conversation until agent completes
- Agent can prompt for interactive input
- Full permission prompts shown to user
- Use for: Interactive work, critical operations

**Background Mode**:
- Runs concurrently with main conversation
- Auto-denies any unpermitted operations
- User gets notification when complete
- Use for: Long-running tasks, parallel research

**Toggle to Background**:
- Press `Ctrl+B` during agent execution
- Agent continues in background
- Main conversation remains responsive

**Example Use Case**:
```
# Start agent in foreground
Use the test-runner agent to run the full test suite

# Press Ctrl+B to background if taking too long
# Continue working while tests run
# Get notified when complete
```

### Resume Subagents

Claude can continue a subagent's work instead of starting fresh:

**How Resume Works**:
- Subagent transcripts stored in `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl`
- Claude detects when user wants to continue previous work
- Agent resumes with full previous context

**Example**:
```
# Initial delegation
Use code-reviewer to review the authentication module

# Later, continue the same agent's work
Continue that code review and analyze the authorization logic

# Claude resumes the code-reviewer agent with full context
```

**Benefits**:
- Preserve agent context across multiple requests
- Build on previous analysis
- Avoid redundant re-reading of files

---

## Examples

### Example 1: dbt Developer Agent

```yaml
---
name: dbt-developer
description: >
  Expert dbt developer for analytics engineering.
  Creates staging, intermediate, and mart models following
  Kimball methodology. Use when building or modifying dbt
  transformations, data models, or SQL-based data pipelines.
tools: Read, Write, Edit, Bash, Glob, Grep
model: sonnet
---

# dbt Developer Agent

You are a senior analytics engineer specializing in dbt and SQL Server.

## Your Role

Build robust, tested data transformation pipelines using dbt.

## Your Expertise

- dbt Core (models, tests, docs, deployment)
- SQL Server (T-SQL, optimization, indexing)
- Dimensional Modeling (star schema, Kimball)
- Data Quality (testing, validation)

## Available Skills

### dbt-runner
Execute dbt commands (run, test, compile, docs)

## Standards

Follow conventions in `claude.md`:
- Naming: stg_, int_, fct_, dim_ prefixes
- SQL style: lowercase keywords, CTEs
- Testing: unique + not_null on PKs
- Documentation: All models in schema.yml

## Workflow

1. Understand requirements
2. Create model files
3. Document in schema.yml
4. Compile to check syntax
5. Run and test models
6. Provide handoff notes

## Success Criteria

- ✅ All models compile without errors
- ✅ All tests pass
- ✅ Documentation complete
- ✅ Code follows standards
- ✅ Handoff info ready
```

### Example 2: Code Reviewer Agent (Official)

From official Claude Code documentation:

```yaml
---
name: code-reviewer
description: Expert code review specialist. Proactively reviews code for quality, security, and maintainability.
tools: Read, Grep, Glob, Bash
model: inherit
---

You are a senior code reviewer ensuring high standards of code quality and security.

When invoked:
1. Run git diff to see recent changes
2. Focus on modified files
3. Begin review immediately

Review checklist:
- Code is clear and readable
- Functions and variables are well-named
- No duplicated code
- Proper error handling
- No exposed secrets or API keys
- Input validation implemented
- Good test coverage
- Performance considerations addressed

Provide feedback organized by priority:
- Critical issues (must fix)
- Warnings (should fix)
- Suggestions (consider improving)
```

### Example 3: Debugger Agent (Official)

From official Claude Code documentation:

```yaml
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behavior.
tools: Read, Edit, Bash, Grep, Glob
model: sonnet
---

You are an expert debugger specializing in root cause analysis.

When invoked:
1. Capture error message and stack trace
2. Identify reproduction steps
3. Isolate the failure location
4. Implement minimal fix
5. Verify solution works
```

### Example 4: Project Manager (Orchestrator)

Frontmatter:

```yaml
---
name: project-manager
description: >
  Senior project manager coordinating dbt, semantic layer,
  and Power BI development. Breaks complex requests into phases,
  delegates to specialists, manages handoffs. Use for end-to-end solutions.
tools: Read, Grep, Glob, Task
model: sonnet
---
```

Body:

```markdown
# Project Manager Agent

You are a senior data platform project manager.

## Your Role

Coordinate specialists to deliver complete data solutions.

## Team of Specialists

- **dbt-developer**: Data modeling and transformations
- **semantic-modeler**: DAX measures and relationships
- **pbi-developer**: Report layouts and visualizations
- **data-qa**: Data quality validation
- **code-reviewer**: Code quality assurance

## Orchestration via Task()

Use Task() tool to spawn subagents (main-thread only — subagents cannot spawn subagents):
```

Python invocation pattern inside the body:

```python
# Sequential execution
Task(
    subagent_type="general-purpose",
    task="Create dbt models using dbt-developer expertise",
    description="Create dbt models"
)

# Wait for completion, then:
Task(
    subagent_type="general-purpose",
    task="Create DAX measures using semantic-modeler expertise",
    description="Create DAX measures"
)
```

Remainder of the body:

```markdown
## Delegation Rules

**Simple Request** (1 agent):
- Direct to specialist without spawning

**Medium Request** (2-3 agents):
- Spawn subagents sequentially or in parallel

**Complex Request** (4+ agents):
- Multi-phase orchestration with handoffs

## Communication Pattern

1. **Assessment**: Analyze scope and plan phases
2. **Execution**: Spawn subagents with clear tasks
3. **Consolidation**: Summarize results and next steps

[Additional sections from project-manager/agent.md...]
```

---

## References

**Official Anthropic Documentation**:
- [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents) - Complete agent creation guide
- [Agent Skills - Claude Code Docs](https://code.claude.com/docs/en/skills) - Skills system documentation
- [Subagents in the SDK - Claude Docs](https://platform.claude.com/docs/en/agent-sdk/subagents) - SDK-level subagent reference
- [Skill authoring best practices - Claude Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) - Official best practices

**Official Anthropic GitHub Repositories**:
- [anthropics/claude-code](https://github.com/anthropics/claude-code) - Main Claude Code repository
- [anthropics/skills](https://github.com/anthropics/skills) - Public repository for Agent Skills
- [anthropics/claude-agent-sdk-demos](https://github.com/anthropics/claude-agent-sdk-demos) - Multi-agent system demos
- [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python) - Python SDK
- [anthropics/claude-quickstarts](https://github.com/anthropics/claude-quickstarts) - Quickstart projects

**Community Best Practices**:
- [Best practices for Claude Code subagents](https://www.pubnub.com/blog/best-practices-for-claude-code-sub-agents/) - Comprehensive guide
- [Subagents in Claude Code: AI Architecture Guide](https://wmedia.es/en/writing/claude-code-subagents-guide-ai) - Architecture patterns
- [ClaudeLog](https://claudelog.com/faqs/what-is-sub-agent-delegation-in-claude-code/) - Delegation documentation

**Community Agent Collections**:
- [VoltAgent - Awesome Claude Code Subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) - 100+ specialized agents
- [wshobson - Agents Repository](https://github.com/wshobson/agents) - Intelligent automation agents
- [lst97 - claude-code-sub-agents](https://github.com/lst97/claude-code-sub-agents) - Full-stack development agents

**GitHub Issues & Discussions**:
- [BUG: Claude Code subagent YAML Frontmatter documentation](https://github.com/anthropics/claude-code/issues/8501) - Frontmatter specifications
- [Feature: Model Parameter for Sub-Agents](https://github.com/anthropics/claude-code/issues/4377) - Model configuration
- [Feature: Proactive Hooks for Command Chaining](https://github.com/anthropics/claude-code/issues/4784) - Hook patterns

---

## Summary

**Agent Creation Checklist**:

1. ✅ Plan the role and responsibilities
2. ✅ Create directory: `Agents/agent-name/`
3. ✅ Write `agent.md` with YAML frontmatter:
   - `name` (required): Unique identifier
   - `description` (required): When to delegate
   - `tools` (optional): Allowed tools
   - `disallowedTools` (optional): Denied tools
   - `model` (optional): sonnet/opus/haiku/inherit
   - `permissionMode` (optional): Permission behavior
   - `skills` (optional): Skills to load
   - `hooks` (optional): Lifecycle hooks
4. ✅ Write comprehensive system prompt
5. ✅ Include standards, workflow, success criteria
6. ✅ Test delegation and functionality
7. ✅ Deploy to appropriate location:
   - `~/.claude/agents/` (all projects)
   - `.claude/agents/` (project-level)
   - `--agents` flag (session-level testing)
8. ✅ Document in project context (claude.md)

**Key Principles**:
- Clear descriptions enable automatic delegation
- Least privilege for tool access
- Single, focused expertise per agent
- Include project context and standards
- Define success criteria
- Test before deployment

---

*This document provides comprehensive guidance for creating Claude Code agents. Always consult the [official documentation](https://code.claude.com/docs/en/sub-agents) for the latest updates and features.*
