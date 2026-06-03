# AGENTS.md — textor

> Guidance for AI coding agents (Claude Code, Cursor, Copilot, etc.) working in this repository.
> Read this before writing any code.

---

## What This Project Is

**textor** is a VSCode extension for anyone who writes prose inside VSCode — engineers writing READMEs and changelogs, technical writers working in docs-as-code workflows, or any writer in `.md`/`.txt` files.

It ships three capability layers:

| Layer | What it does |
|---|---|
| **Spellcheck** | Real-time spell diagnostics using `nspell` + `dictionary-en` |
| **Thesaurus** | Offline synonym lookup via right-click context menu → inline replace |
| **AI writing** | Rephrase/rewrite selections, tone/clarity scoring, contextual synonym suggestions |

**Key principle**: the extension works offline first. AI features require provider configuration but must never be mandatory.

---

## Repository Layout

```
src/
  extension.ts          # Entry point. Registers all providers and commands.
  ai/
    index.ts            # Provider abstraction — the interface all implementations must satisfy
    claude.ts           # Claude (Anthropic) implementation
    openai.ts           # OpenAI implementation
    ollama.ts           # Ollama (local) implementation
  config/
    index.ts            # Reads + exposes VS Code extension settings
  spellchecker/
    index.ts            # Registers DiagnosticCollection, wires document listeners
    diagnostics.ts      # Spell-checks a document, returns Diagnostic[]
    dictionary.ts       # Loads .aff/.dic data for a given language code
  thesaurus/
    index.ts            # Registers the context menu command + CodeActionProvider
    hover.ts            # HoverProvider — shows synonyms on hover (secondary UX)
    offline.ts          # Offline synonym lookup engine (no AI, no network)
  types/
    dictionary-en.d.ts  # Type declarations for the `dictionary-en` package
```

`dist/` — compiled output (webpack). Never edit directly.

---

## Architecture Decisions

### AI Provider Abstraction

All AI features go through a single interface defined in `src/ai/index.ts`. The concrete provider (Claude, OpenAI, Ollama) is resolved at runtime based on user settings, with an auto-detect fallback. The interface shape is **not finalized yet** — do not solidify it without explicit design discussion.

When implementing, the interface should cover at minimum:

- `rephrase(text: string, context: string): Promise<string[]>` — returns alternative phrasings
- `score(text: string): Promise<ClarityScore>` — tone/clarity analysis
- `synonyms(word: string, context: string): Promise<string[]>` — contextual synonyms (AI-powered, richer than offline)

### Thesaurus UX

Primary interaction: **right-click → context menu → replace word**. This is implemented as a `CodeActionProvider` (kind `refactor`), not a command palette entry.

The `HoverProvider` (`hover.ts`) is a secondary surface — it may show a quick synonym preview on hover, but the replace action lives in the context menu.

### File Activation

Default activation: `.md` and `.txt`. Users can extend this via a setting (`textor.languageIds`). Use `activationEvents` + `onLanguage` in `package.json`, and guard all document listeners with a language ID check derived from config.

---

## Hard Constraints — Do Not Violate

These are non-negotiable. Do not work around them.

### 1. No telemetry or data collection

User text **must never leave the machine** without explicit, per-feature user consent. This means:

- No analytics, crash reporting, or usage tracking of any kind.
- AI provider calls are opt-in and require the user to configure an API key or local model endpoint.
- Never log document content, even to the Output channel. Log metadata only (word count, language ID, etc.).

### 2. No new dependencies without asking

The bundle size matters for install speed and security surface. Before adding any `npm` dependency:

- Check if the VSCode API or Node.js built-ins already cover the need.
- If a dep is genuinely needed, state the reason and ask the user before installing.
- Dev dependencies (types, test tooling) are lower bar but still warrant a mention.

### 3. No UI outside VSCode native APIs

Do not introduce webviews, custom sidebars, or any HTML/CSS UI unless explicitly requested. Approved surfaces:

- `DiagnosticCollection` (spell errors)
- `HoverProvider` (thesaurus on hover)
- `CodeActionProvider` (right-click menu actions)
- `QuickPick` / `InputBox` (for commands requiring user input)
- `OutputChannel` (for debug/status logging)
- Status bar items (for provider status)

If a feature seems to need a webview, stop and discuss before implementing.

---

## Development Workflow

### Build

```bash
npm run compile       # webpack dev build
npm run watch         # webpack watch mode (use during development)
npm run package       # production build (minified, no source map)
```

### Lint

```bash
npm run lint          # eslint on src/
```

### Test

```bash
npm run pretest       # compiles tests + source + lints
npm test              # runs vscode-test suite
```

Tests live in `src/test/`. Use the VSCode test infrastructure (`@vscode/test-electron`) — not Jest or Mocha standalone. Extension tests run inside a real VSCode host process.

### Debug

Use `F5` in VSCode with the `Run Extension` launch config (`.vscode/launch.json`). This opens an Extension Development Host.

---

## Coding Conventions

### TypeScript

- Strict mode is on (`tsconfig.json`). No `any` escapes.
- Prefer explicit return types on exported functions.
- Use `async/await`; avoid raw Promise chains.
- Dispose of VSCode resources. Every provider registration returns a `Disposable` — push it to `context.subscriptions`.

### VSCode Extension Patterns

- All provider registrations happen in `extension.ts` `activate()`. Module `index.ts` files export a single `register(context)` function.
- Deactivation cleanup happens via `context.subscriptions` — don't implement `deactivate()` unless you have resources that VSCode can't clean up automatically.
- Use `vscode.workspace.onDidChangeTextDocument` for live spell checking. Debounce it — do not check on every keystroke.
- Use `vscode.workspace.getConfiguration('textor')` via the config module. Never read config directly in feature modules.

### Error Handling

- AI provider errors must be surfaced as dismissible notifications (`vscode.window.showErrorMessage`), not crashes.
- Dictionary load failures should degrade gracefully — disable spell checking for that session and log to the Output channel.
- Never throw unhandled promise rejections.

---

## Current State

Most files are empty stubs. Here is what exists vs. what needs building:

| File | State | Notes |
|---|---|---|
| `extension.ts` | Scaffold only | Replace hello-world with real registration |
| `spellchecker/dictionary.ts` | Working | Loads en dictionary; extend for other languages later |
| `spellchecker/diagnostics.ts` | Empty | Implement with `nspell` |
| `spellchecker/index.ts` | Empty | Register DiagnosticCollection + document listeners |
| `thesaurus/offline.ts` | Empty | Offline synonym engine (no dep yet — discuss first) |
| `thesaurus/hover.ts` | Empty | HoverProvider |
| `thesaurus/index.ts` | Empty | Register CodeActionProvider for right-click → replace |
| `ai/index.ts` | Empty | Define provider interface |
| `ai/claude.ts` | Empty | Claude implementation |
| `ai/openai.ts` | Empty | OpenAI implementation |
| `ai/ollama.ts` | Empty | Ollama implementation |
| `config/index.ts` | Empty | Settings reader |

Build order recommendation: **spellchecker → thesaurus (offline) → config → AI abstraction → AI providers**.

---

## Open Questions (Do Not Assume — Ask)

- Offline thesaurus data source: no dependency is currently installed. Options include `moby-thesaurus`, a bundled JSON wordlist, or a local lookup table. Ask before picking one.
- AI provider interface: the shape above is a starting proposal, not final. Confirm before implementing.
- Language support beyond English: dictionary.ts has a `lang` param but only `en` is wired. Do not add other languages speculatively.
- `package.json` `contributes` section needs commands, menus, and configuration schema — do not add these without aligning on UX first.
