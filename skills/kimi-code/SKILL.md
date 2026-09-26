---
name: kimi-code
description: How this repository's rules and reference skills map onto Kimi Code CLI — the rules arrive as system prompt from the plugin, the three skills are invoked with the Skill tool, and the file names differ from Claude Code's. Loaded automatically at session start by this plugin's sessionStart entry.
whenToUse: Loaded at session start whenever this plugin is enabled; also useful whenever a rule from CLAUDE.md names a Claude Code mechanism and the equivalent Kimi Code mechanism is what actually applies.
---

# This plugin, in Kimi Code CLI

The rules in `CLAUDE.md` at the plugin root are **injected into the system prompt**
(`systemPromptPath`), so they are in force for the whole session. Do not re-read the
file to "load" them, and do not treat the injection as a summary — it is the text.

## The three reference skills

`nextjs`, `react-router` and `tauri` are ordinary Agent Skills here. The mandates in
`CLAUDE.md` (`Skill("nextjs")` before writing Next.js code, `Skill("react-router")`
before React Router code, `Skill("tauri")` before Tauri IPC code) translate directly:
invoke the Skill tool with that name before touching the corresponding code.

## Names that differ from Claude Code

| `CLAUDE.md` says | Kimi Code equivalent |
| --- | --- |
| `CLAUDE.md` (project instructions) | `AGENTS.md` in the project root; project-local config in `.kimi-code/` |
| `CLAUDE.md` (global instructions) | `~/.agents/AGENTS.md` (cross-tool) or `$KIMI_CODE_HOME/AGENTS.md` |
| `.claude/skills/`, `~/.claude/skills/` | `.kimi-code/skills/`, `~/.kimi-code/skills/`, `~/.agents/skills/`, or a directory listed in `extra_skill_dirs` |
| the skill listing's `Skill(...)` call | the `Skill` tool, same names |

## Rules that need a Kimi-side note

- **Commit messages**: no `@` characters (the file's text covers the whole message;
  the subject line is where it actually bites), and no AI attribution trailers.
- **Confirmation before modification**: a plan is not authorization. Wait for an
  explicit go-ahead before writing, deleting or creating any file.
- **Stack scoping**: the Rust/Tauri rules apply only where Rust is present, the React
  rules only where React is present. A rule whose stack is absent is not a question
  to ask — it simply does not apply.
- **Skills and MCP outrank this document** where they cover the same ground, and a
  conflict is to be flagged rather than silently resolved.
