# skills

Personal engineering configuration by [@LikeDreamwalker](https://github.com/LikeDreamwalker) —
**dual-installable**: Claude Code reads `.claude-plugin/`, Kimi Code CLI reads
`kimi.plugin.json`. One copy of the rules and the skills serves both.

**CLAUDE.md** — non-negotiable engineering rules. Always loaded, always enforced.

**Skills** — canonical reference implementations. Loaded on demand via
MANDATORY directives in CLAUDE.md.

## Skills

| Skill | Covers |
|---|---|
| `react-router` | loader, action, Form, useFetcher, useNavigation, errorElement, async states, client boundaries, TypeScript |
| `nextjs` | Server Components, Server Actions, useActionState, loading.tsx, error.tsx, not-found.tsx, Suspense, streaming, client boundaries, TypeScript |
| `tauri` | IPC command signatures, thiserror enums, State\<T\>, spawn_blocking, reqwest, IPC type alignment, integration tests |
| `kimi-code` | Kimi Code CLI specifics: which file names and mechanisms replace Claude Code's, how the three skills above are invoked there (auto-loaded at session start) |

Rust language guidance is delegated to [rust-skills](https://github.com/actionbook/rust-skills).

## Install

### Claude Code

```bash
# Add as a plugin marketplace, then install
claude plugins marketplace add LikeDreamwalker/skills
claude plugins install engineering-skills
```

Or clone directly and symlink:

```bash
git clone https://github.com/LikeDreamwalker/skills.git ~/.claude/skills-config
ln -sf ~/.claude/skills-config/CLAUDE.md ~/.claude/CLAUDE.md
```

### Kimi Code CLI

The same repository is a Kimi plugin: `kimi.plugin.json` at the root declares the
rules file as system prompt and the same skills directory.

```
/plugins install https://github.com/LikeDreamwalker/skills
/reload
```

A local checkout works the same way: `/plugins install <path-to-this-repo>`.

- `systemPromptPath: ./CLAUDE.md` — the rules are injected into the system prompt
  while the plugin is enabled, so they are always in force rather than loaded on
  demand.
- `skills: ./skills/` — Kimi's `SKILL.md` format is the same `name` + `description`
  frontmatter Claude Code uses, so the skills are not duplicated for the two tools.
- `sessionStart.skill: kimi-code` — loads `skills/kimi-code/SKILL.md` at session
  start, which maps the Claude-Code-flavoured names in CLAUDE.md onto Kimi Code's
  (`AGENTS.md`, `~/.agents/AGENTS.md`, `extra_skill_dirs`, the `Skill` tool).

Prefer wiring it by hand instead of by plugin:

```toml
# ~/.kimi-code/config.toml  — top-level key, must sit above the first [table]
extra_skill_dirs = ["C:/path/to/skills/skills"]
```

```bash
# global rules, the cross-tool location Kimi Code also reads
cp CLAUDE.md ~/.agents/AGENTS.md
```

Do one or the other, not both: the plugin already injects CLAUDE.md as system
prompt and registers the same four skills, and wiring both paths would load the
rules twice.

Requires the [rust-skills](https://github.com/actionbook/rust-skills) plugin.

## Structure

```
skills/
  .claude-plugin/
    marketplace.json
    plugin.json
  kimi.plugin.json
  CLAUDE.md
  skills/
    react-router/SKILL.md
    nextjs/SKILL.md
    tauri/SKILL.md
    kimi-code/SKILL.md
  README.md
```

## License

MIT
