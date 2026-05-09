# CLAUDE.md

This file describes the `design.md` repository for AI assistants working in this codebase.

## Repository Overview

`design.md` is a Google open-source project that defines the **DESIGN.md format** — a plain-text specification for describing a visual identity (design system) to coding agents. It ships a CLI toolkit (`@google/design.md`) for linting, diffing, and exporting DESIGN.md files.

The DESIGN.md format combines:
- **YAML front matter** — machine-readable design tokens (colors, typography, spacing, etc.)
- **Markdown body** — human-readable rationale organized into `##` sections

## Monorepo Structure

```
design.md/
├── packages/
│   └── cli/                     # @google/design.md npm package
│       ├── src/
│       │   ├── index.ts         # CLI entrypoint (citty)
│       │   ├── version.ts       # VERSION constant
│       │   ├── utils.ts         # Shared CLI utilities
│       │   ├── commands/        # CLI subcommands
│       │   │   ├── lint.ts
│       │   │   ├── diff.ts
│       │   │   ├── export.ts
│       │   │   └── spec.ts
│       │   └── linter/          # Core library (also exported as @google/design.md/linter)
│       │       ├── index.ts     # Public API exports
│       │       ├── lint.ts      # Top-level lint() function
│       │       ├── parser/      # Markdown + YAML → ParsedDesignSystem
│       │       ├── model/       # ParsedDesignSystem → DesignSystemState (resolved tokens)
│       │       ├── linter/      # Lint rules + runner
│       │       │   └── rules/   # Individual lint rules (one file per rule)
│       │       ├── tailwind/    # Tailwind export handler
│       │       ├── dtcg/        # W3C DTCG export handler
│       │       ├── fixer/       # Auto-fix handler (section order)
│       │       └── spec-gen/    # Generates docs/spec.md from spec.mdx + spec-config.ts
│       └── dist/                # Build output (not committed)
├── docs/
│   └── spec.md                  # Generated spec — do NOT edit directly
├── examples/                    # Sample DESIGN.md files
│   ├── atmospheric-glass/
│   ├── paws-and-paths/
│   └── totality-festival/
├── turbo.json                   # Turborepo task pipeline
├── package.json                 # Root workspace (Bun workspaces)
└── tsconfig.base.json           # Shared TypeScript config
```

## Toolchain

| Tool | Purpose |
|------|---------|
| **Bun** | Package manager, test runner, bundler |
| **Turborepo** | Monorepo task orchestration |
| **TypeScript** | Strict mode, ES2022 target, ESNext modules |
| **citty** | CLI framework for subcommands |

## Development Commands

Run from the repo root unless noted:

```bash
# Install dependencies
bun install

# Run all tests
bun test

# Build the CLI package
cd packages/cli && bun run build

# Run the CLI directly (no build needed)
bun run cli lint examples/atmospheric-glass/DESIGN.md

# Regenerate docs/spec.md (after editing spec-gen/spec.mdx or spec-config.ts)
cd packages/cli && bun run spec:gen

# Check the published package structure
cd packages/cli && bun run check-package
```

## Data Flow

```
DESIGN.md file
     │
     ▼
ParserHandler          (packages/cli/src/linter/parser/handler.ts)
     │  ParsedDesignSystem  (raw YAML + section list)
     ▼
ModelHandler           (packages/cli/src/linter/model/handler.ts)
     │  DesignSystemState   (resolved tokens, symbol table, color luminance)
     ▼
LinterRunner           (packages/cli/src/linter/linter/runner.ts)
     │  LintResult          (Finding[], GradedTokenEdits)
     ▼
CLI commands / programmatic API
```

## Linting Pipeline

Each lint rule lives in `packages/cli/src/linter/linter/rules/` as a pair of files:
- `<rule-name>.ts` — rule implementation (`LintRule` function)
- `<rule-name>.test.ts` — Bun tests

Active rules (in order of severity):

| Rule | Severity | Description |
|------|----------|-------------|
| `broken-ref` | error | Unresolvable `{token.ref}` references |
| `missing-primary` | warning | No `primary` color defined |
| `missing-typography` | warning | Colors exist but no typography tokens |
| `contrast-ratio` | warning | Component bg/text pairs below WCAG AA (4.5:1) |
| `orphaned-tokens` | warning | Color tokens never referenced by any component |
| `section-order` | warning | Sections out of canonical order |
| `missing-sections` | info | Optional sections absent |
| `token-summary` | info | Count of tokens per section |

All rules are registered in `packages/cli/src/linter/linter/rules/index.ts`.

## Public API

The package exports two entry points:

```typescript
// Main CLI (bin)
import { lint } from '@google/design.md/linter';

// Full programmatic API
import {
  lint,
  runLinter,
  preEvaluate,
  DEFAULT_RULES,
  TailwindEmitterHandler,
  DtcgEmitterHandler,
  fixSectionOrder,
  contrastRatio,
} from '@google/design.md/linter';
```

## TypeScript Conventions

- **Strict mode** is enforced: `strict: true`, `exactOptionalPropertyTypes: true`, `noUncheckedIndexedAccess: true`.
- All source files use ESM (`"type": "module"`). Import paths end with `.js` (e.g., `import ... from './spec.js'`).
- Each subsystem follows a **handler/spec pattern**:
  - `spec.ts` — types, interfaces, and pure validation helpers
  - `handler.ts` — class implementing the spec interface
  - `handler.test.ts` — Bun tests for the handler
- Handlers **never throw** — all errors are returned as typed result objects.
- `LinterHandler` is deprecated; use `runLinter()` and `preEvaluate()` directly.

## Adding a New Lint Rule

1. Create `packages/cli/src/linter/linter/rules/<rule-name>.ts` exporting a `LintRule`.
2. Create the matching `<rule-name>.test.ts`.
3. Add the rule to `DEFAULT_RULES` in `packages/cli/src/linter/linter/rules/index.ts`.
4. Export it from `packages/cli/src/linter/index.ts`.
5. Add an entry to the rules table in `docs/spec.md` (or run `bun run spec:gen`).

## The `spec.md` File

`docs/spec.md` is **generated** from:
- `packages/cli/src/linter/spec-gen/spec.mdx` — the spec prose (MDX)
- `packages/cli/src/linter/spec-gen/spec-config.ts` — rule metadata

Do **not** edit `docs/spec.md` directly. Run `cd packages/cli && bun run spec:gen` to regenerate it.

## CI

GitHub Actions (`.github/workflows/test.yml`) runs on every push and PR:
1. `bun install`
2. `bun test` (all packages)
3. `cd packages/cli && bun run build`
4. Node.js smoke test against a real DESIGN.md example
5. Tarball smoke test (`npm pack` + install + lint + programmatic import)

## License

Apache 2.0. All source files must carry the Google LLC copyright header. New files require the standard Apache 2.0 license header block.
