# CLAUDE.md — VS Code (Code - OSS)

## Build Commands

```bash
# Install dependencies (Node 24.18.0 required — see .nvmrc)
nvm use
npm install

# Compile the workbench (client + Copilot)
npm run compile

# Compile built-in extensions
npm run compile-extensions

# Compile web version
npm run compile-web

# Compile CLI
npm run compile-cli

# Fast incremental build with extensions
npm run build-fast

# Watch mode (recompiles on change)
npm run watch           # client
npm run watch-client    # client only
npm run watch-extensions # extensions only
npm run watch-web       # web only

# Full CI build (compile + hygiene + type checks)
npm run core-ci
npm run extensions-ci

# Type check only (no emit)
npm run typecheck-client
```

> **Environment note:** Electron version is pinned to 42.7.0 via `.npmrc`. The `.nvmrc` pins Node to 24.18.0. Use `nvm use` before `npm install`.

---

## Test Commands

```bash
# Run Node.js unit tests
npm run test-node

# Run browser unit tests (Playwright)
npm run test-browser

# Run integration tests (Linux/macOS — requires display)
npm run test-extension

# Smoke tests
npm run smoketest

# Run a single test file (Node)
./scripts/test.sh path/to/test/file.test.js

# Run a single named test (Mocha --grep)
./scripts/test.sh --grep "test name or regex"

# Browser integration tests
./scripts/test-web-integration.sh

# Integration tests
./scripts/test-integration.sh
```

Test infrastructure: **Mocha** for unit/integration tests, **Playwright** for browser and smoke tests. Tests live under `test/unit/node/`, `test/unit/browser/`, and `test/smoke/`.

---

## Development Commands

```bash
# Launch the Electron app (development)
./scripts/code.sh          # Linux/macOS
./scripts/code.bat         # Windows

# Launch in web browser mode
./scripts/code-web.sh

# Launch as a server
./scripts/code-server.sh

# Launch agent host
./scripts/code-agent-host.sh

# Gulp tasks (via npm run / npx gulp)
npx gulp hygiene          # Run code hygiene checks
npx gulp eslint           # Run ESLint via Gulp
npx gulp copy-codicons    # Copy icon assets
```

VS Code debug configurations and tasks are in `.vscode/launch.json` and `.vscode/tasks.json`.

---

## Project Structure

```
.
├── .devcontainer/          # Dev container configuration
├── .eslint-plugin-local/   # Custom local ESLint rules
├── .github/                # GitHub Actions CI workflows
│   └── workflows/
├── .vscode/                # Workspace settings, launch/task configs, MCP
├── build/                  # Gulp build system, scripts, Azure Pipelines config
│   ├── gulpfile.*.ts       # Platform-specific Gulp tasks
│   ├── lib/                # Build utilities (cyclic deps, policies, builtins)
│   ├── checker/            # Architecture layer validation
│   ├── agent-sdk/          # Claude/Codex agent SDK integration
│   └── azure-pipelines/    # Azure DevOps pipeline definitions
├── cli/                    # Rust-based CLI implementation
├── extensions/             # Built-in VS Code extensions (one dir per extension)
├── remote/                 # Remote development packages
├── resources/              # Platform-specific resources (icons, completions)
│   ├── darwin/
│   ├── linux/
│   ├── win32/
│   └── server/
├── scripts/                # Developer utility scripts (launch, test runners)
├── src/                    # Main TypeScript/JavaScript source code
│   ├── main.ts             # Electron main entry
│   ├── cli.ts              # CLI entry
│   ├── server-main.ts      # Server entry
│   ├── bootstrap-*.ts      # Platform bootstrap files
│   └── vs/                 # Core VS Code modules (workbench, platform, etc.)
└── test/                   # Test suites
    ├── unit/node/          # Node.js unit tests (Mocha)
    ├── unit/browser/       # Browser unit tests (Playwright)
    └── smoke/              # End-to-end smoke tests
```

---

## Architecture Notes

### Layered Architecture
VS Code enforces strict layer separation validated in CI (`build/checker/`). The main layers are:

- **`vs/base`** — Platform-agnostic utilities (no DOM, no Node.js APIs)
- **`vs/platform`** — Shared services with dependency injection
- **`vs/editor`** — The Monaco editor core
- **`vs/workbench`** — The full IDE shell, composed from platform services
- **`vs/code`** — Electron desktop entry and host-specific code

Imports must only flow downward: `workbench` → `editor/platform` → `base`. Violations are caught by the layer checker. Cyclic dependency detection also runs in CI.

### Dependency Injection
Services are registered via a central IoC container (`vs/platform/instantiation`). Constructors declare service dependencies with decorators; the container resolves them at startup. Avoid direct instantiation of services.

### Extension Architecture
Built-in extensions in `extensions/` are first-class but isolated from core via the extension host process. Each extension has its own `package.json` and may have a separate tsconfig. The product manifest (`product.json`) controls which built-in extensions are bundled.

### Build System
The build system is **Gulp 4** (entry: `gulpfile.mjs` → `build/gulpfile.ts`). Platform-specific tasks are in `build/gulpfile.{desktop,web,reh,...}.ts`. Bundling uses esbuild 0.27.2 for the web/server; the desktop build uses a Gulp pipeline.

### Key Dependencies
| Package | Purpose |
|---|---|
| Electron 42.7.0 | Desktop shell |
| TypeScript (v6) | Primary language |
| xterm.js | Integrated terminal |
| node-pty | Terminal PTY |
| @anthropic-ai/sdk | Claude AI integration |
| Playwright | Browser automation / testing |
| Mocha | Unit/integration test runner |
| ripgrep (binary) | File search backend |

### Multi-Platform Targets
The codebase targets four runtime environments: **Electron desktop**, **browser (web)**, **remote server (REH)**, and **CLI (Rust)**. Code under `src/vs/` often contains platform-specific implementations (e.g., `electron-main/`, `browser/`, `node/`, `common/`) — use the correct target.

---

## Code Style

### TypeScript
- **Tab indentation** (size 4) — enforced by `.editorconfig` and `tsfmt.json`
- **No `var`** — use `const`/`let`
- **No `eval`**
- Spaces after commas and semicolons; spaces around binary operators
- No extra spaces inside parentheses or brackets
- Function keyword spacing: `function foo()` (not `function(`)

### ESLint
Configuration: `eslint.config.js` (flat config format, ESLint 9).

Run linting:
```bash
npm run eslint
npm run stylelint   # CSS/SCSS
```

Notable enforced rules:
- **License headers** on all source files (`eslint-plugin-header`)
- **No unexternalized strings** — UI strings must go through the localization layer (`nls.localize`)
- **No const enums** — use regular enums or string unions
- **Import order** enforced via `eslint-plugin-import`
- Custom VS Code–specific rules in `.eslint-plugin-local/`

### Naming Conventions
- `PascalCase` for classes, interfaces, enums, and types
- `camelCase` for variables, functions, and methods
- `_camelCase` prefix for private class members
- `I` prefix for service interfaces (e.g., `IFileService`)
- File names: `camelCase.ts` for modules, `PascalCase.ts` for classes

### Import Ordering
Imports follow ESLint `import` plugin rules. Group: external packages first, then internal `vs/` paths. No default exports in core code — use named exports.

### JSDoc
Only document non-obvious public APIs. `eslint-plugin-jsdoc` is active; avoid redundant comments that restate the type signature.
