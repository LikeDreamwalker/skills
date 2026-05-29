# skills

Personal Claude Code configuration by [@LikeDreamwalker](https://github.com/LikeDreamwalker).

**CLAUDE.md** — non-negotiable engineering rules. Always loaded, always enforced.

**Skills** — canonical reference implementations. Loaded on demand via
MANDATORY directives in CLAUDE.md.

## Skills

| Skill | Covers |
|---|---|
| `react-router` | loader, action, Form, useFetcher, useNavigation, errorElement, async states, client boundaries, TypeScript |
| `nextjs` | Server Components, Server Actions, useActionState, loading.tsx, error.tsx, not-found.tsx, Suspense, streaming, client boundaries, TypeScript |
| `tauri` | IPC command signatures, thiserror enums, State\<T\>, spawn_blocking, reqwest, IPC type alignment, integration tests |

Rust language guidance is delegated to [rust-skills](https://github.com/actionbook/rust-skills).

## Install

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

Requires the [rust-skills](https://github.com/actionbook/rust-skills) plugin.

## Structure

```
skills/
  .claude-plugin/
    marketplace.json
    plugin.json
  CLAUDE.md
  skills/
    react-router/SKILL.md
    nextjs/SKILL.md
    tauri/SKILL.md
  README.md
```

## License

MIT
