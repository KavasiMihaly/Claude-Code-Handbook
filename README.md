# Claude Code Handbook

Canonical guides for building **agents, skills, hooks, and plugins** with Claude Code. Distilled from Anthropic's docs, real-world plugin authoring (the [OneDayBI Marketplace](https://github.com/KavasiMihaly/AI-plugins) plugins), and the community.

By [Mihaly Kavasi](https://github.com/KavasiMihaly) — companion handbook to the [OneDayBI Marketplace](https://github.com/KavasiMihaly/AI-plugins) and [Claude Code Essentials](https://github.com/KavasiMihaly/Claude-Code-Essentials).

---

## Why this handbook exists

The official Claude Code docs cover *what* the primitives are. This handbook covers **how to actually build with them** — the patterns, anti-patterns, and gotchas that show up only after shipping something real and watching it break.

Each topic has two files:
- A **how-to** — step-by-step build instructions, the minimum viable structure, common mistakes.
- A **best-practices** — patterns, multi-agent orchestration, frontmatter reference, external source citations, and (where relevant) verbatim findings from plugins that broke in production.

---

## Contents

### How-to guides — learn the mechanics

| Guide | What's in it |
|---|---|
| [`how-to-create-agents.md`](how-to-create-agents.md) | Custom agents (`agent.md`), Task tool, model selection, isolation modes, master-clone patterns, agent teams |
| [`how-to-create-skills.md`](how-to-create-skills.md) | `SKILL.md` structure, scripts vs markdown, progressive disclosure, description as routing signal |
| [`how-to-create-hooks.md`](how-to-create-hooks.md) | All 9+ hook events, JSON protocol, plugin-level vs settings-level registration, exit-code semantics |
| [`how-to-create-plugins.md`](how-to-create-plugins.md) | Plugin manifest, packaging, marketplace publishing, plugin-internal hooks, MCP servers in plugins |

### Best practices — learn the patterns

| Doc | What's in it |
|---|---|
| [`claude-code-agent-best-practices.md`](claude-code-agent-best-practices.md) | Multi-agent orchestration, frontmatter reference, model dispatch, scoping rules, external-source validation |
| [`claude-code-skill-best-practices.md`](claude-code-skill-best-practices.md) | Progressive disclosure, skill composition, plugin patterns, when to use a skill vs an agent |
| [`claude-code-hooks-best-practices.md`](claude-code-hooks-best-practices.md) | Plugin-level hooks, event selection, observability, atomic-Bash rule, hook design pitfalls |
| [`claude-code-plugin-best-practices.md`](claude-code-plugin-best-practices.md) | 16 anti-patterns from real plugin work, verbatim quote library, community video references, audit gates |

### Quick reference

| Doc | What's in it |
|---|---|
| [`claude-code-commands-and-settings.md`](claude-code-commands-and-settings.md) | The slash commands worth knowing (`/context`, `/compact`, `/agents`, `/permissions`…), the `settings.json` precedence hierarchy, how to wire up MCP servers, plugin config & control (`enabledPlugins`, `pluginConfigs`, marketplace allow/deny lists), and the config keys that have no UI (permissions, hooks, env, statusLine) |

### Templates

| File | What it is |
|---|---|
| [`templates/CLAUDE.md.template`](templates/CLAUDE.md.template) | Opinionated starter for `~/.claude/CLAUDE.md` — universal rules marked **[UNIVERSAL]**, personal preferences marked **[OPINION]** so you keep what fits and swap the rest. |

---

## How to use this handbook

**New to Claude Code?** Start with the four how-to guides in order: agents → skills → hooks → plugins. Each builds on the previous one. Then read the matching best-practices doc when you're ready to ship something real.

**Already shipping?** Jump to the four best-practices docs. They assume you know the primitives and focus on the patterns that hold up under stress.

**Setting up a new machine?** Copy [`templates/CLAUDE.md.template`](templates/CLAUDE.md.template) into `~/.claude/CLAUDE.md` and customise it.

---

## Companion projects

- [**OneDayBI Marketplace**](https://github.com/KavasiMihaly/AI-plugins) — Claude Code plugin marketplace. Add with `/plugin marketplace add KavasiMihaly/AI-plugins`.
- [**Claude Code Essentials**](https://github.com/KavasiMihaly/Claude-Code-Essentials) — The hooks and statusline described in the templates. Installs as a plugin.
- [**dbt Pipeline Toolkit**](https://github.com/KavasiMihaly/DBT-Pipeline-Plugin) — Plugin whose findings (F1–F10) underpin many of the patterns in `claude-code-plugin-best-practices.md`.
- [**Fabric Dataflow Migration**](https://github.com/KavasiMihaly/Dataflow-to-Notebook-Plugin) — Plugin whose findings (N1–N17) extend those patterns to multi-stage agent orchestration.

---

## Contributing

Found a pattern, anti-pattern, or gotcha worth documenting? Open an issue or PR. Real-world breakage is the primary source of value here — concrete reproductions and post-mortems are especially welcome.

---

## License

MIT — see [LICENSE](LICENSE).

---

## Author

**Mihaly Kavasi** — [@KavasiMihaly](https://github.com/KavasiMihaly) | [OneDayBI](https://www.onedaybi.com) | [Self-Service BI Blog](https://selfservicebi.co.uk)
