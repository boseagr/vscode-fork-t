# VS Code (Code OSS) — Developer Reference

## Build Commands

```bash
# Install dependencies
npm install

# Full compilation (client + extensions)
npm run compile

# Compile only the client
npm run compile-client

# Transpile only (no type checking — fastest)
npm run transpile-client

# Type check only (no emit)
npm run typecheck-client

# Watch mode (continuous recompilation on file changes)
npm run watch

# Watch client only
npm run watch-client

# Watch extensions only
npm run watch-extensions

# Fast build for development
npm run build-fast
```

Output is written to the `out/` directory.

## Test Commands

```bash
# Full unit test suite (runs inside Electron)
./scripts/test.sh

# Unit tests with debug mode
./scripts/test.sh --debug

# Single test file
npm run test-node -- --run <filepath>

# Single named test (by glob pattern)
./scripts/test.sh --glob "<test name pattern>"

# Browser tests (Playwright)
npm run test-browser -- --browser chromium --browser webkit

# Integration tests
./scripts/test-integration.sh

# Extension tests
npm run test-extension

# Smoke tests (automated UI)
npm run smoketest
```

Test locations:
- `test/unit/` — unit tests (Mocha)
- `test/integration/` — API/integration tests
- `test/smoke/` — automated UI tests
- `test/sanity/` — release sanity tests

## Development Commands

```bash
# Run the editor from compiled source
./scripts/code.sh

# Run as a web-only version
./scripts/code-web.sh

# Run as a remote server
./scripts/code-server.sh

# Hygiene / pre-commit checks
npm run hygiene
npm run eslint
npm run stylelint
npm run valid-layers-check
npm run precommit
```

## Project Structure

```
.
├── src/                    # Main TypeScript source code
│   └── vs/
│       ├── base/           # Core utilities: events, lifecycle, disposables, data structures
│       ├── code/           # Electron bootstrapping and main process entry point
│       ├── editor/         # Monaco editor core (syntax, editing, rendering)
│       ├── platform/       # Abstract service definitions and cross-cutting implementations
│       ├── server/         # Remote development server
│       ├── sessions/       # Session management
│       └── workbench/      # Main UI shell, views, panels, and editor features
├── extensions/             # ~98 built-in extensions (language support, git, debugger, etc.)
├── build/                  # Gulp-based build system and compilation scripts
├── test/                   # All test suites (unit, integration, smoke, sanity)
├── cli/                    # Rust-based `code` CLI implementation
├── scripts/                # Shell/batch scripts for running, testing, and profiling
├── remote/                 # Remote development support files
├── resources/              # Static assets, icons, and platform-specific resources
├── .github/workflows/      # CI/CD pipeline definitions
├── eslint.config.js        # ESLint flat config (ESLint 9)
├── src/tsconfig.base.json  # Shared TypeScript compiler options
└── package.json            # npm scripts, devDependencies
```

## Architecture Notes

**Layered architecture** — strict layer separation is enforced by `valid-layers-check`. Higher layers may depend on lower ones, but not vice versa:

1. `base` — zero-dependency utilities, data structures, event emitters, disposables
2. `platform` — abstract service interfaces + implementations (107+ sub-modules)
3. `editor` — Monaco editor core, independent of VS Code shell
4. `code` — Electron main/renderer process bootstrapping
5. `workbench` — full UI shell wiring all services and views together
6. `server` — remote server entry point

**Service container / dependency injection** — services are registered and resolved via the platform DI container. New services must declare an `IServiceBrand` and register via `registerSingleton` or equivalent.

**Disposable pattern** — all resources that need cleanup implement `IDisposable`. Use `DisposableStore` or `MutableDisposable` for composite/swappable lifetimes. Leaking disposables causes memory leaks tracked by `debugModel`.

**Event-driven** — internal communication uses typed `Emitter<T>` / `Event<T>` from `vs/base/common/event`. Prefer `Event.once`, `Event.filter`, and `Event.map` combinators over raw subscriptions.

**Module system** — AMD (transitioning to ESM). Imports within `src/` use paths relative to `src/vs/` without file extensions.

**Major runtime dependencies:**
- Electron 42.7.0 (shell host)
- TypeScript 6.0.2 (compiled with `--experimental-strip-types`)
- XTerm.js (integrated terminal)
- Monaco Editor (embedded editor core)
- vscode-textmate + oniguruma (grammar/syntax highlighting)
- SQLite3 (vscode-sqlite3, used by some extensions)
- Playwright (browser and smoke tests)
- Mocha (unit test runner)

## Code Style

**TypeScript** (`src/tsconfig.base.json`):
- Target: ES2024, module: nodenext
- `strict: true`, `noUnusedLocals: true`, `noImplicitReturns: true`, `noFallthroughCasesInSwitch: true`
- Experimental decorators enabled
- No `any` casts without justification

**Formatting** (`tsfmt.json`):
- Indent: 4 spaces (tabs in configuration, spaces in TS)
- Space after commas and semicolons
- Spaces around binary operators
- No spaces inside parentheses or brackets

**ESLint** (`eslint.config.js`, flat config, ESLint 9):
- Custom local plugin at `.eslint-plugin-local/` enforces VS Code-specific rules:
  - No unexternalized strings (NLS strings must use `nls.localize`)
  - No `declare const` enums in platform-facing APIs
  - No dangerous type assertions
  - Service brand declarations required
  - Parameter property visibility must be explicit
- No `eval`, no `Buffer()` constructor
- Run `npm run eslint` to check; fix with `npm run eslint -- --fix`

**TSEC** (TypeScript security checker) also runs in CI to detect unsafe patterns.

**Naming conventions:**
- Interfaces prefixed with `I` (e.g., `IEditorService`)
- Abstract classes and base implementations use `Abstract` prefix or `Base` suffix
- Private members use `_` prefix by convention in some older files, but new code avoids it
- Event properties named `onDid*` / `onWill*`
