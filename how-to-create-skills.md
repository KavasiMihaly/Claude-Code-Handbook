# How to Create Claude Code Skills

**Last Updated**: January 2026
**Purpose**: Step-by-step guide for creating effective Claude Code skills for data engineering workflows

---

## Table of Contents

1. [What are Skills?](#what-are-skills)
2. [File Structure](#file-structure)
3. [SKILL.md Anatomy](#skillmd-anatomy)
4. [Design Principles](#design-principles)
5. [Step-by-Step Creation Process](#step-by-step-creation-process)
6. [Bundled Resources](#bundled-resources)
7. [Best Practices](#best-practices)
8. [Examples](#examples)
9. [Validation & Testing](#validation--testing)
10. [Common Patterns](#common-patterns)

---

## What are Skills?

**Skills** are modular, self-contained packages that extend Claude's capabilities with:
- Specialized workflows
- Tool integrations
- Domain expertise
- Bundled resources (scripts, references, assets)

**Key Difference from Agents**:
- **Agents** = AI personas with specific expertise and behavior patterns
- **Skills** = Reusable tools/capabilities that agents can invoke

---

## File Structure

A skill is a **folder** containing a required SKILL.md file and optional bundled resources:

```
skill-name/
├── SKILL.md              # REQUIRED - Only required file
├── scripts/              # Optional - Executable code
│   ├── helper.py
│   └── runner.sh
├── references/           # Optional - Documentation loaded as needed
│   ├── api-reference.md
│   └── examples.md
└── assets/               # Optional - Templates, images, fonts
    └── template.json
```

### Directory Placement

**User-level skills** (portable across all projects):
```
~/.claude/skills/
├── dbt-runner/
├── sql-formatter/
└── tabular-editor/
```

**Project-level skills** (specific to this project):
```
<project-root>/.claude/skills/
├── custom-validator/
└── project-specific-tool/
```

**Development/staging** (before copying to ~/.claude/skills/):
```
<repo-root>/Skills/
├── dbt-runner/
├── sql-formatter/
└── tabular-editor/
```

---

## SKILL.md Anatomy

The SKILL.md file consists of **two required parts**:

### Part 1: YAML Frontmatter (Required)

```yaml
---
name: skill-name-here
description: Clear description of what this skill does and when to use it. Include triggers and contexts.
---
```

> **Optional fields.** `name` and `description` are the only required keys. SKILL.md frontmatter also supports `argument-hint`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `model`, `effort`, `context`, `agent`, `hooks`, and `paths`. See `claude-code-skill-best-practices.md` § 5 ("SKILL.md Frontmatter Reference") for each field's semantics and the full enum values.

#### Validation Rules

**name**:
- Maximum 64 characters
- Lowercase letters, numbers, and hyphens only
- No spaces, no underscores
- No XML tags
- No reserved words
- Examples: `dbt-runner`, `sql-formatter`, `tabular-editor`

**description**:
- Maximum 1024 characters
- Non-empty
- No XML tags
- **Most critical field** - determines when Claude triggers the skill
- Should include:
  - What the skill does
  - When to use it (triggers/contexts)
  - Key capabilities

#### Good vs. Bad Descriptions

**❌ Bad**:
```yaml
description: Runs dbt commands
```

**✅ Good**:
```yaml
description: Execute dbt commands (run, test, compile, docs) for analytics engineering workflows. Use when running dbt models, testing data quality, generating documentation, or managing dbt projects. Supports model selection, full refresh, and slim CI patterns.
```

### Part 2: Markdown Body (Required)

Instructions that tell Claude how to use the skill:

```markdown
# Skill Name

Brief overview of what this skill does.

## When to Use

- Use case 1
- Use case 2
- Use case 3

## Available Operations

### Operation 1
How to perform operation 1...

### Operation 2
How to perform operation 2...

## Examples

Example 1:
\`\`\`bash
python scripts/example.py --param value
\`\`\`

## Guidelines

- Guideline 1
- Guideline 2

## Reference Documentation

For detailed information, see:
- [API Reference](references/api.md)
- [Advanced Usage](references/advanced.md)
```

#### Keep SKILL.md Concise

- **Target**: Under 500 lines / <5k words
- **Why**: Minimize context window consumption
- **How**: Move detailed content to `references/` files

---

## Design Principles

### 1. Progressive Disclosure (Most Important)

Skills use a **three-level loading system**:

**Level 1: Metadata** (Always in context)
- Just the `name` and `description` from YAML frontmatter
- ~100 tokens total
- Always loaded, determines if skill should trigger

**Level 2: SKILL.md Body** (When skill triggers)
- Loaded when Claude determines the skill is relevant
- Keep under 500 lines
- High-level guidance and navigation

**Level 3: Bundled Resources** (As needed)
- References, scripts, assets loaded only when Claude needs them
- Effectively unlimited size
- Scripts execute without loading content into context

### 2. Concise is Key

**Remember**: Claude is already very smart. Only add context Claude doesn't have:
- ✅ Tool-specific syntax and commands
- ✅ Project-specific configurations
- ✅ Specialized domain knowledge
- ❌ General programming concepts
- ❌ Common best practices Claude already knows

### 3. Set Appropriate Degrees of Freedom

**High Freedom** (Text-based instructions):
- For tasks with multiple valid approaches
- When Claude should adapt to context
- Example: "Design a data quality test suite"

**Medium Freedom** (Pseudocode/scripts with parameters):
- For semi-structured tasks
- When there's a recommended approach but variations are acceptable
- Example: "Use this script template but adapt as needed"

**Low Freedom** (Specific scripts):
- For fragile, error-prone operations
- When there's one correct way to do something
- Example: "Run exactly: `python scripts/deploy.py --env prod`"

---

## Step-by-Step Creation Process

### Step 1: Identify the Need

Ask yourself:
- What specialized task needs automation?
- What domain knowledge should be bundled?
- What tools or workflows should be integrated?
- Will this be reused across projects?

**Example**: "We frequently run dbt commands with specific patterns. A dbt-runner skill would standardize this."

### Step 2: Plan the Contents

Identify what resources are needed:

| Resource Type | Purpose | Example |
|---------------|---------|---------|
| **Scripts** | Executable automation | `scripts/run_dbt.py` |
| **References** | Detailed documentation | `references/commands.md` |
| **Assets** | Templates, boilerplate | `assets/model-template.sql` |

### Step 3: Create the Directory Structure

```bash
mkdir -p skill-name/{scripts,references,assets}
touch skill-name/SKILL.md
```

### Step 4: Write SKILL.md

1. **Start with YAML frontmatter**:
   ```yaml
   ---
   name: skill-name
   description: What it does and when to use it
   ---
   ```

2. **Write the markdown body**:
   - Overview
   - When to use
   - Available operations
   - Examples
   - Links to reference files

3. **Keep it concise**: Move details to `references/`

### Step 5: Create Bundled Resources

**Scripts** (scripts/):
- Python, Bash, or any executable code
- Self-contained utilities
- Accept parameters for flexibility
- Return JSON or structured output when possible

**References** (references/):
- API documentation
- Detailed examples
- Configuration guides
- Troubleshooting tips

**Assets** (assets/):
- Templates
- Configuration files
- Images or diagrams
- Boilerplate code

### Step 6: Test the Skill

1. Copy skill folder to `~/.claude/skills/`
2. Start Claude Code session
3. Test skill triggering by describing relevant tasks
4. Verify Claude loads and uses the skill correctly
5. Check that scripts execute properly

### Step 7: Iterate and Improve

- Monitor how Claude uses the skill
- Add missing documentation
- Improve descriptions for better triggering
- Add more examples based on real usage
- Optimize for clarity and conciseness

---

## Bundled Resources

### Scripts Directory (scripts/)

**Purpose**: Executable code that performs specific operations

**Characteristics**:
- Executed **without loading content into context**
- Only output consumes tokens
- Perfect for deterministic, reusable tasks

**Examples**:
```python
# scripts/run_dbt.py
import subprocess
import sys
import json

def run_dbt(command, select=None, full_refresh=False):
    cmd = ['dbt', command]
    if select:
        cmd.extend(['--select', select])
    if full_refresh:
        cmd.append('--full-refresh')

    result = subprocess.run(cmd, capture_output=True, text=True)

    return {
        'success': result.returncode == 0,
        'stdout': result.stdout,
        'stderr': result.stderr
    }

if __name__ == '__main__':
    # Parse arguments and execute
    result = run_dbt(sys.argv[1], ...)
    print(json.dumps(result))
```

### References Directory (references/)

**Purpose**: Documentation loaded into context only when needed

**When to Use**:
- Detailed API documentation
- Extensive examples
- Schema definitions
- Troubleshooting guides

**Progressive Disclosure Pattern**:

In SKILL.md:
```markdown
## Advanced Features

For complex transformations, see:
- [Incremental Models](references/incremental.md)
- [Custom Materializations](references/materializations.md)
- [Macro Development](references/macros.md)
```

**Keep references one level deep**: Don't have reference files that link to other reference files.

### Assets Directory (assets/)

**Purpose**: Files used in skill output

**Examples**:
- Template files (SQL, Python, configuration)
- Images or logos for reports
- Boilerplate code
- Fonts for document generation

```
assets/
├── templates/
│   ├── staging-model.sql
│   ├── fact-model.sql
│   └── dimension-model.sql
└── config/
    └── dbt_project_template.yml
```

---

## Best Practices

### Description Writing

**Include in Description**:
- ✅ What the skill does
- ✅ When to use it (triggers)
- ✅ Key capabilities
- ✅ Context clues for triggering

**Examples**:

(See the ✅ Good dbt-runner example in [SKILL.md Anatomy → Good vs. Bad Descriptions](#good-vs-bad-descriptions) above for the canonical pattern.)

**sql-formatter**:
```yaml
description: Format SQL code using SQLFluff for SQL Server/T-SQL dialect. Use when formatting SQL files, enforcing SQL style standards, or fixing SQL linting errors. Applies consistent indentation, keyword casing, and formatting rules.
```

**tabular-editor**:
```yaml
description: Execute Tabular Editor CLI commands for Power BI semantic model operations. Use when creating DAX measures, managing relationships, running Best Practice Analyzer, or deploying semantic model changes via XMLA.
```

### SKILL.md Body Writing

**Use imperative/infinitive form**:
- ✅ "Run the script with..."
- ✅ "Execute dbt test to validate..."
- ❌ "You should run..."
- ❌ "The user can execute..."

**Organize by task, not by file**:
- ✅ "## Creating Models" → links to relevant scripts/references
- ❌ "## Scripts" → just lists files

**Link to references for details**:
```markdown
## Incremental Models

For large fact tables, use incremental materialization.

Basic example:
\`\`\`sql
{{ config(materialized='incremental') }}
select * from source
{% if is_incremental() %}
  where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
\`\`\`

For advanced patterns, see [references/incremental-strategies.md](references/incremental-strategies.md)
```

### Script Writing

**Make scripts self-contained**:
```python
#!/usr/bin/env python3
"""
Script description

Usage:
    python script.py <arg1> <arg2>

Example:
    python script.py run --select fct_sales
"""

# All imports at top
# Clear argument parsing
# JSON output for structured results
# Proper error handling
```

**Return structured output**:
```python
result = {
    'success': True,
    'message': 'Operation completed',
    'data': {...},
    'errors': []
}
print(json.dumps(result, indent=2))
```

### What NOT to Include

**Don't include**:
- ❌ README.md (use SKILL.md instead)
- ❌ INSTALLATION_GUIDE.md (include in SKILL.md if needed)
- ❌ CHANGELOG.md (not needed for skills)
- ❌ CONTRIBUTING.md (skills are simple, not full projects)
- ❌ LICENSE files (unless required for bundled code)

**Only include**:
- ✅ SKILL.md
- ✅ Scripts that execute operations
- ✅ References with detailed documentation
- ✅ Assets used in output

---

## Examples

### Example 1: Simple Skill (Script Only)

```
dbt-runner/
├── SKILL.md
└── scripts/
    └── run_dbt.py
```

**SKILL.md**:
```yaml
---
name: dbt-runner
description: Execute dbt commands (run, test, compile, docs) for analytics engineering. Use when running dbt models, testing data quality, or generating documentation.
---

# dbt Runner

Execute dbt commands in the current project.

## Usage

Run all models:
\`\`\`bash
python scripts/run_dbt.py run
\`\`\`

Run specific model:
\`\`\`bash
python scripts/run_dbt.py run --select model_name
\`\`\`

Run tests:
\`\`\`bash
python scripts/run_dbt.py test
\`\`\`

Available commands: run, test, compile, docs, clean
```

### Example 2: Skill with References

```
sql-formatter/
├── SKILL.md
├── scripts/
│   └── format_sql.py
└── references/
    ├── rules.md
    └── examples.md
```

**SKILL.md**:
```yaml
---
name: sql-formatter
description: Format SQL code using SQLFluff for T-SQL dialect. Use when formatting SQL files or enforcing SQL style standards.
---

# SQL Formatter

Format SQL files with SQLFluff using T-SQL dialect.

## Quick Usage

\`\`\`bash
python scripts/format_sql.py path/to/file.sql
\`\`\`

## Formatting Rules

See [references/rules.md](references/rules.md) for complete formatting rules.

## Examples

See [references/examples.md](references/examples.md) for before/after examples.
```

### Example 3: Skill with Assets

```
dbt-template-generator/
├── SKILL.md
├── scripts/
│   └── generate_model.py
└── assets/
    └── templates/
        ├── staging.sql
        ├── intermediate.sql
        └── fact.sql
```

**SKILL.md**:
```yaml
---
name: dbt-template-generator
description: Generate dbt model files from templates following project conventions. Use when creating new dbt staging, intermediate, or fact models.
---

# dbt Template Generator

Generate dbt model files from standardized templates.

## Usage

Create staging model:
\`\`\`bash
python scripts/generate_model.py staging customers --source erp
\`\`\`

Create fact model:
\`\`\`bash
python scripts/generate_model.py fact sales
\`\`\`

Templates include:
- Standard naming conventions
- Config blocks
- Testing scaffolding
- Documentation stubs
```

---

## Validation & Testing

### Manual Validation

**Check YAML frontmatter**:
- ✅ name: lowercase, hyphens only, max 64 chars
- ✅ description: clear, includes triggers, max 1024 chars

**Check file structure**:
- ✅ SKILL.md exists
- ✅ No README.md or other unnecessary files
- ✅ Scripts are executable
- ✅ References are one level deep

**Check SKILL.md body**:
- ✅ Under 500 lines
- ✅ Uses imperative form
- ✅ Includes examples
- ✅ Links to references for details

### Testing the Skill

1. **Copy to skills directory**:
   ```bash
   cp -r skill-name ~/.claude/skills/
   ```

2. **Start Claude Code session**

3. **Test triggering**:
   - Describe a task the skill should handle
   - Verify Claude mentions/uses the skill
   - Check skill loads correctly

4. **Test execution**:
   - Verify scripts execute without errors
   - Check outputs are correct
   - Confirm references load when needed

5. **Iterate**:
   - Improve description if skill doesn't trigger
   - Add missing documentation
   - Fix script errors
   - Enhance examples

---

## Common Patterns

### Pattern 1: Command Wrapper

**Use Case**: Wrap command-line tools with standardized parameters

**Structure**:
```
tool-wrapper/
├── SKILL.md
└── scripts/
    └── run_tool.py
```

**Example**: dbt-runner, sql-formatter, tabular-editor

### Pattern 2: Template Generator

**Use Case**: Generate files from templates

**Structure**:
```
generator/
├── SKILL.md
├── scripts/
│   └── generate.py
└── assets/
    └── templates/
        └── template.ext
```

**Example**: dbt model generator, DAX measure generator

### Pattern 3: Knowledge Base

**Use Case**: Provide domain-specific knowledge and examples

**Structure**:
```
knowledge-base/
├── SKILL.md
└── references/
    ├── domain-guide.md
    ├── examples.md
    └── api-reference.md
```

**Example**: BigQuery schema reference, API documentation

### Pattern 4: Workflow Orchestrator

**Use Case**: Multi-step workflows with decision points

**Structure**:
```
workflow/
├── SKILL.md
├── scripts/
│   ├── step1.py
│   ├── step2.py
│   └── orchestrate.py
└── references/
    └── workflow-guide.md
```

**Example**: Dashboard build workflow, data quality workflow

---

## Skills inside Plugins

Standalone skills (under `~/.claude/skills/` or `<project>/.claude/skills/`) and plugin-shipped skills are loaded by the same engine, but their **path resolution rules differ**. A skill that works in dev mode can break silently once shipped inside a plugin unless you follow the rules below.

See `claude-code-skill-best-practices.md` § 6 ("Skill Patterns from Shipped Plugins") for the underlying rationale and failure-mode catalog.

### Reference scripts and templates with explicit roots

| Context | Use this variable | Why |
|---|---|---|
| Inside `SKILL.md` referencing files in the same skill | `${CLAUDE_SKILL_DIR}/scripts/<file>.py` | Resolves to the skill's own folder regardless of dev vs installed |
| Inside an `agent.md` referencing a skill shipped by the same plugin | `${CLAUDE_PLUGIN_ROOT}/skills/<name>/scripts/<file>.py` | Plugin-relative path that survives plugin moves |
| Inside a hook command in `plugin.json` | `${CLAUDE_PLUGIN_ROOT}/hooks/<file>.py` | Same — never use `~` or absolute user paths |

**Avoid:** bare relative paths like `python scripts/run.py` and tilde paths like `~/.claude/skills/...`. Both work in dev mode and break on a fresh plugin install.

### Remap plugin userConfig to env vars

When a plugin exposes `userConfig` keys (e.g. `fabric_workspace_id`), the harness passes them as `CLAUDE_PLUGIN_OPTION_<KEY>`. A common pattern is to add a small loader at the top of any Python script that reads these vars:

```python
# Reproduced from claude-code-skill-best-practices.md § 6.3
import os

def _load_plugin_userconfig_env() -> None:
    """Promote CLAUDE_PLUGIN_OPTION_* vars to plain names for scripts."""
    prefix = "CLAUDE_PLUGIN_OPTION_"
    for key, val in os.environ.items():
        if key.startswith(prefix):
            short = key[len(prefix):]
            os.environ.setdefault(short, val)

_load_plugin_userconfig_env()
```

This lets the same script run standalone (with `FABRIC_WORKSPACE_ID=...` in the shell) or inside a plugin (with `CLAUDE_PLUGIN_OPTION_FABRIC_WORKSPACE_ID=...`) without code changes.

### "Locked by plugin" UI label

When a skill is shipped by a plugin, the Claude Code UI shows a small **"locked by plugin"** label next to it. This means:

- Users cannot edit the skill in-place (changes would be lost on plugin update).
- To customize, users must fork the plugin or copy the skill to `~/.claude/skills/<name>/` (which takes precedence).
- Skill version is tied to the plugin version — bump `plugin.json#version` whenever the skill changes.

### Atomic Bash rule

Every shell command surfaced by a plugin-shipped skill (in `SKILL.md`, in any script that shells out, or in any hook the plugin registers) **must be a single atomic operation** — no `&&`, `||`, `;`, `|`, subshells, `$(...)`, or heredocs. Compound shell expressions fall through to interactive permission prompts, which block background subagents silently.

Rewrite compound expressions as either:
- Multiple sequential tool calls, or
- A Python script that does the composition and is itself invoked atomically.

See `claude-code-plugin-best-practices.md` § 5 for the full rationale (this rule was the root cause of one of the most expensive `dbt-pipeline-toolkit` bugs).

---

## Summary Checklist

When creating a skill, ensure:

**Structure**:
- [ ] Created skill folder with descriptive name
- [ ] SKILL.md exists and is properly formatted
- [ ] YAML frontmatter is valid (name, description)
- [ ] No unnecessary files (README, CHANGELOG, etc.)

**Content**:
- [ ] Description clearly states what and when to use
- [ ] SKILL.md body is under 500 lines
- [ ] Imperative/infinitive writing style
- [ ] Examples included
- [ ] Links to references for details

**Resources**:
- [ ] Scripts are executable and self-contained
- [ ] Scripts return structured output (JSON)
- [ ] References are one level deep
- [ ] Assets are properly organized

**Testing**:
- [ ] Copied to ~/.claude/skills/
- [ ] Tested triggering in Claude Code
- [ ] Verified script execution
- [ ] Confirmed reference loading
- [ ] Iterated based on testing

---

## Additional Resources

**Official Documentation**:
- [Agent Skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Skill Authoring Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [How to Create Custom Skills](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills)

**Examples**:
- [anthropics/skills GitHub Repository](https://github.com/anthropics/skills)
- [Awesome Claude Skills](https://github.com/travisvn/awesome-claude-skills)

**Guides**:
- [Inside Claude Code Skills - Mikhail Shilkov](https://mikhail.io/2025/10/claude-code-skills/)
- [Claude Skills and CLAUDE.md Guide](https://www.gend.co/blog/claude-skills-claude-md-guide)

---

**Last Updated**: May 2026
**Related Documents**:
- [Architecture guide](../../_Documentation/claude-code-agentic-architecture-guide.md)
- [Project CLAUDE.md](../../CLAUDE.md)
- [Skills best practices](claude-code-skill-best-practices.md) — companion best-practices doc
- [How to create plugins](how-to-create-plugins.md) — packaging skills into shippable plugins
