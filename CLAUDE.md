# System Instructions

## Defaults
- Logic lives in Rust by default. React runtime is opt-in: use only when
  browser APIs or local UI state genuinely require it.
- All code is production code. There is no prototype mode, no throwaway mode,
  no "just to see if it works." Every line you write must meet the full
  standard below. Shortcuts, stubs, and half-implementations are not options.

## Skills & MCP
- When a loaded skill or MCP tool covers the topic at hand (Rust error
  handling, React Router patterns, shadcn/ui component API, Next.js
  conventions, etc.), its guidance takes priority over this document's
  general rules.
- If a skill or MCP tool provides a specific pattern that conflicts with a
  rule here, the skill/MCP wins. Flag the conflict so the rule can be
  reviewed.
- Never write from memory what a skill or MCP tool can provide. Invoke the
  tool, get the authoritative answer, then act on it.

## Pre-flight (every task)
1. Read `CLAUDE.md`, `AGENTS.md`, and any index files in the project root.
2. Verify existing conventions, component library, directory structure, and
   installed dependencies before proposing new patterns or libraries.
3. Do not guess.

## Uncertainty
- When a project convention conflicts with a rule in this document, ask —
  don't assume which takes priority.
- When a rule's boundary is unclear, flag it. Don't guess.

## React / Frontend

### Data fetching
- Route-level data MUST use React Router `loader`/`action` or Next.js Server
  Components with async `fetch`.
- `useState` + `useEffect` for data fetching is FORBIDDEN. If you can get it
  from a loader, server component, or server action, do that.

### Client boundary
- `useState`, `useEffect`, `useRef`, and event handlers are RESERVED for
  leaf-level interactivity: form inputs, toggles, drag-and-drop, media controls.
- Layouts, cards, containers, page skeletons MUST remain server-renderable.
- Pattern: push state down to the smallest leaf that actually needs it, not up
  to the nearest layout.

### Async completeness
Every route and every component that depends on async data MUST handle four
states explicitly:
- **Loading** — skeleton or spinner (prefer `loading.tsx` in Next.js,
  `useNavigation().state === "loading"` in React Router)
- **Empty** — "no items" illustration with clear user guidance
- **Error** — `ErrorBoundary` at the nearest layout level + route-level
  `error.tsx` or `errorElement`
- **Success** — the actual UI

Unhandled promise rejections or missing error boundaries are FORBIDDEN.

### TypeScript
- Explicit return types on all function signatures and interface contracts.
- Prefer discriminated unions and `enum` (or `as const` objects) over `string`.
- `any` is FORBIDDEN. Use `unknown` + type narrowing when the shape is genuinely
  unknowable at the boundary.

MANDATORY: Before writing React Router or Next.js code, invoke the corresponding skill:
- React Router → `Skill("react-router")`
- Next.js → `Skill("nextjs")`

## Rust / Tauri

### Logic ownership
- All business logic, state machines, and side-effect orchestration MUST live
  in Rust. JS is limited to UI rendering and user input capture.
- Non-presentational logic pushed to JS is FORBIDDEN.

### IPC contracts
- Every `#[tauri::command]` MUST declare explicit input types and a return type
  of `Result<T, CustomError>`.
- Export TypeScript-compatible type definitions for every IPC boundary.
- Implicit or untyped payloads (`serde_json::Value`, `HashMap<String, Value>`)
  crossing IPC are FORBIDDEN.

### Error handling
- `unwrap` and `expect` are FORBIDDEN in production code paths.
- All errors MUST convert into a structured `Result<T, CustomError>` variant
  that the frontend can parse without string-matching.
- Recommended: `thiserror` derive macro for error enum definitions.

### Async discipline
- Blocking operations (audio, file I/O, network, heavy computation) MUST run in
  `async` commands or `tauri::async_runtime::spawn`.
- Calling blocking code on the Tauri main thread is FORBIDDEN.
- Pattern: `tokio::task::spawn_blocking` for CPU-bound work, async I/O via
  `tokio::fs` / `reqwest`.

### State ownership
- Long-lived resources (audio sessions, file handles, device connections) MUST
  be managed via Tauri `State<T>`.
- The frontend references resources by opaque string IDs only.
- Passing raw handles, file descriptors, or pointers across IPC is FORBIDDEN.

MANDATORY: Before writing Tauri IPC code, invoke `Skill("tauri")`.
For Rust language-level guidance (ownership, error handling, concurrency,
async, etc.), always consult rust-skills first. The `tauri` skill covers
Tauri-specific IPC patterns only.

## Security
- User input crossing IPC MUST be validated in Rust before any filesystem,
  network, or system operation.
- File paths from the frontend MUST be canonicalized and bounds-checked in Rust
  before use. Relative path traversal (`../`) MUST be rejected.
- Secrets, tokens, and API keys MUST live in Rust-managed state. Exposing them
  to the JS runtime is FORBIDDEN.

## UI / Styling

### Component reuse
- Use existing project components before creating anything new. Read the
  project's component directory to know what's already available.
- Recreating existing primitives (buttons, cards, inputs, modals, dropdowns) is
  FORBIDDEN.

### shadcn/ui
- All shadcn/ui components MUST be added via `npx shadcn add <component>` run
  through Bash. Never write shadcn component source from memory.
- Hand-written components that resemble shadcn patterns but bypass the CLI are
  FORBIDDEN. The CLI is the single source of truth for shadcn code in the
  project.
- Before adding a new shadcn component, check the project's component directory
  to confirm it doesn't already exist.

### Tailwind
- Use design tokens from the Tailwind scale (`w-8`, `p-4`, `text-sm`).
- Arbitrary values (`w-[128px]`) and raw `px` are FORBIDDEN. The only exception
  is non-scale values (e.g., `w-[7px]`) where no Tailwind token reasonably
  approximates the design intent, and only with explicit user approval.
- Every `px` value below design-token threshold MUST convert to the nearest
  Tailwind token:

  | px | token | | px | token | | px | token |
  |----|-------|-|----|-------|-|----|-------|
  | 4px | 1 | | 20px | 5 | | 48px | 12 |
  | 8px | 2 | | 24px | 6 | | 64px | 16 |
  | 12px | 3 | | 32px | 8 | | 80px | 20 |
  | 16px | 4 | | 40px | 10 | | 96px | 24 |
- Use existing Tailwind config tokens and CSS variables from `tailwind.config`
  before defining new values.

### Responsive
- Every layout MUST implement `sm` / `md` / `lg` / `xl` breakpoints in
  mobile-first order.
- Non-responsive layouts are FORBIDDEN.

  | prefix | min-width |
  |--------|-----------|
  | (base) | 0 |
  | sm | 640px |
  | md | 768px |
  | lg | 1024px |
  | xl | 1280px |
  | 2xl | 1536px |

### i18n
- Detection: if `package.json` includes `next-i18next`, `react-intl`,
  `i18next`, `lingui`, or if a `locales/` / `messages/` directory exists,
  the project has i18n configured.
- When i18n is detected: ALL user-facing strings MUST be written as translation
  keys with corresponding entries in the locale files. Hardcoded strings in any
  language are FORBIDDEN.
- When i18n is NOT detected: use the project's primary language consistently.

## Testing

- New Tauri commands MUST include integration tests.
- New UI components with conditional rendering (loading/empty/error/success)
  MUST include unit tests covering each state.
- Bug fixes MUST include a regression test.
- Frontend: use the project's test framework (Vitest, Jest, or Playwright)
- Rust: `cargo test` with `#[cfg(test)]` modules

## Engineering Discipline

### Always follow user direction
- Execute user commands exactly as specified. Do not expand scope, add unrequested
  features, or make changes beyond what the user asked for.
- Never modify code, commit, push, or create PRs without explicit user approval.
- When the user says to stop, discuss, or wait — stop immediately. Do not continue
  implementing until alignment is confirmed.
- If a request is ambiguous, ask for clarification rather than assuming.

### No black boxes
- Enumerate types, states, status codes, and error variants explicitly.
- If a domain cannot be fully enumerated from existing code, flag the gap and
  ask — do not hide it behind `string`, `any`, or catch-all variants.

### Verify before proposing
- Check existing files, configs, installed crates, and `package.json` before
  introducing new dependencies, utility libraries, or architectural patterns.
- Do not assume a library exists just because it's popular.


## Summary of Prohibitions
| Category | Forbidden |
|----------|-----------|
| React | `useEffect`-driven data fetching, `any`, client state in layouts, missing error/loading/empty states |
| Rust | `unwrap`/`expect`, untyped IPC payloads, blocking on main thread, raw handle passing to JS |
| UI | Recreating existing components, manual shadcn source, `px` units, non-responsive layouts |
| Security | Secrets in JS runtime, unvalidated IPC input, unchecked file paths from frontend |
| All | Prototypes and shortcuts, guessing conventions without reading index files, unverified dependencies |
