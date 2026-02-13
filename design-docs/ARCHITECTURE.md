# StyForge — Technical Architecture

> **Audience:** Developers contributing to or extending StyForge  
> **Prerequisite:** Read DESIGN.md first for the "why." This document covers the "how."

---

## Overview

StyForge is a TypeScript CLI tool that compiles a YAML configuration file into a plain CSS file. Its architecture is a modular, staged pipeline inspired by compiler design principles.

```
YAML Config
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                    CONFIG STAGE                      │
│  Load → Parse → Validate (Zod) → Resolve defaults  │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                    TOKEN STAGE                       │
│  Expand scales → Resolve refs → Build token tree    │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                    THEME STAGE                       │
│  Map semantic tokens → Generate theme CSS blocks    │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                   MODULE STAGE                       │
│  Execute each module: reset, base, components, util  │
└─────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                   OUTPUT STAGE                       │
│  Assemble → Wrap in @layers → Minify? → Write disk  │
└─────────────────────────────────────────────────────┘
    │
    ▼
Plain CSS File(s)
```

---

## Stage 1: Config Stage

**Location:** `src/core/config/`

### Responsibilities
- Load `styforge.yaml` (or a custom path passed via CLI flag)
- Parse YAML into a raw JavaScript object
- Validate the raw object against the Zod schema
- Merge with the full default config tree (so unspecified keys always have values)
- Resolve token cross-references (e.g. `"{tokens.colors.primary.500}"` → `"#3B82F6"`)

### Key Files

**`loader.ts`** — Reads the YAML file from disk. Returns raw parsed object.

```typescript
import { readFileSync } from 'fs';
import { parse } from 'yaml';

export function loadConfig(configPath: string): unknown {
  const raw = readFileSync(configPath, 'utf-8');
  return parse(raw);
}
```

**`validator.ts`** — Validates with Zod. Throws structured errors on failure.

The validator does not just return `true/false`. Validation errors are formatted into human-readable messages with line numbers (via `yaml` library's position tracking) and fix suggestions.

**`defaults.ts`** — The complete default config. Every key that exists in the schema exists here with a sensible default. This is what gets merged with the user's config.

Merging strategy: deep merge, with user values taking precedence. Arrays are replaced entirely (not merged), because YAML arrays represent complete lists of intent (e.g., a spacing scale is not additive — it's a complete replacement).

**`resolver.ts`** — Walks the validated config and resolves all `{token.ref}` strings. This is a two-pass operation:
- Pass 1: Collect all token values into a flat lookup map.
- Pass 2: Walk every string value in the config. If it matches the `{...}` pattern, replace with the resolved value.

Circular reference detection is mandatory. A reference that resolves to another reference that resolves back to the first will be caught and reported with a clear error.

---

## Stage 2: Token Stage

**Location:** `src/core/tokens/`

### Responsibilities
- Expand shorthand scale definitions into full token maps
- Compute derived values (e.g. color shade generation from a base color)
- Produce a fully-resolved, flat-accessible token tree

### Scale Generation

Scales are one of the most powerful features. Instead of listing every value, developers describe a pattern:

**Spacing scales:**
```yaml
spacing:
  base: 4
  preset: tailwind  # Generates: 1→4px, 2→8px, 3→12px, 4→16px, 5→20px, 6→24px...
```

Built-in presets:
- `tailwind` — The standard 4px-based scale (1→4px, 2→8px, 3→12px, 4→16px, 5→20px, 6→24px, 8→32px, 10→40px, 12→48px, 16→64px...)
- `compact` — Denser: base 4px with smaller increments
- `spacious` — More breathing room: base 8px with larger increments  
- `fibonacci` — 4, 8, 12, 20, 32, 52, 84... (Fibonacci sequence × base)

**Typography scales (modular scale):**
```yaml
typography:
  sizes:
    base: 16
    scale: major-third  # Ratio: 1.250
```

Built-in modular scales:
- `minor-second` — ratio 1.067
- `major-second` — ratio 1.125
- `minor-third` — ratio 1.200
- `major-third` — ratio 1.250 (recommended — balanced for UI)
- `perfect-fourth` — ratio 1.333 (good for editorial)
- `golden` — ratio 1.618 (dramatic hierarchy)

Generated sizes from `major-third` base 16px:
- `xs` → 10.24px → `0.64rem`
- `sm` → 12.8px → `0.8rem`
- `base` → 16px → `1rem`
- `lg` → 20px → `1.25rem`
- `xl` → 25px → `1.563rem`
- `2xl` → 31.25px → `1.953rem`
- `3xl` → 39.06px → `2.441rem`
- `4xl` → 48.83px → `3.052rem`

**Color palette generation:**
```yaml
colors:
  primary:
    generate: true
    base: "#3B82F6"
    steps: 10  # 50, 100, 200, 300, 400, 500, 600, 700, 800, 900
```

Color generation uses OKLCH (perceptually uniform color space) to produce shades that look correct to the human eye, rather than the jagged jumps you get from simple HSL lightness adjustments.

---

## Stage 3: Theme Stage

**Location:** `src/core/tokens/themes.ts`

### Responsibilities
- Take the resolved token tree and the themes config
- For each theme, build a complete map of semantic token → resolved value
- Generate the CSS custom property blocks for each theme

### Output Pattern

```css
/* Generated by theme stage */

/* Default tokens (always in :root) */
:root {
  /* Raw tokens */
  --color-primary-500: #3B82F6;
  --spacing-4: 1rem;
  
  /* Semantic tokens — light theme (default) */
  --color-background: #FFFFFF;
  --color-surface: #F9FAFB;
  --color-text-primary: #111827;
  --color-accent-primary: #2563EB;
}

/* Dark theme — activated by data attribute */
[data-theme="dark"] {
  --color-background: #030712;
  --color-surface: #111827;
  --color-text-primary: #F9FAFB;
  --color-accent-primary: #60A5FA;
}

/* Dark theme — activated by OS preference (when no manual override) */
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --color-background: #030712;
    --color-surface: #111827;
    --color-text-primary: #F9FAFB;
    --color-accent-primary: #60A5FA;
  }
}
```

Note: The `:root:not([data-theme="light"])` pattern is critical. It means:
- OS dark → dark theme (unless user has explicitly chosen light)
- OS light → light theme (unless user has explicitly chosen dark)
- `data-theme="dark"` → always dark regardless of OS
- `data-theme="light"` → always light regardless of OS

---

## Stage 4: Module Stage

**Location:** `src/modules/`

This is the heart of the extensibility system.

### The Module Interface

```typescript
interface StyForgeModule {
  /**
   * Unique module identifier.
   * Convention: "phase/name" e.g. "components/button"
   */
  name: string;

  /**
   * Which @layer this module's output belongs to.
   * Determines where in the CSS file the output is placed.
   */
  phase: 'tokens' | 'reset' | 'base' | 'components' | 'utilities';

  /**
   * Whether this module should run given the current config.
   * If false, the module is skipped entirely.
   */
  isEnabled(config: StyForgeConfig): boolean;

  /**
   * Generate CSS. Returns a string of raw CSS content.
   * The pipeline wraps this in the appropriate @layer block.
   */
  generate(context: BuildContext): string;
}
```

### The BuildContext

The `BuildContext` is the shared state object passed to every module. It is immutable from the module's perspective — modules read from it but cannot write to it.

```typescript
interface BuildContext {
  /** The fully resolved configuration */
  config: StyForgeConfig;

  /** 
   * The resolved token tree.
   * All {token.refs} have been resolved. All scales expanded.
   * This is what modules use to access values.
   */
  tokens: ResolvedTokens;

  /**
   * Semantic token maps per theme.
   * Use these (not raw tokens) in component CSS.
   */
  themes: ResolvedThemes;

  /**
   * Convenience: access resolved CSS custom property names
   * e.g. tokens.var('spacing.4') → 'var(--spacing-4)'
   */
  t: TokenAccessor;

  /** Logger for debug/warning output */
  logger: Logger;
}
```

### The TokenAccessor (`context.t`)

This is a DX convenience that makes module authoring cleaner:

```typescript
// Instead of writing this in modules:
`var(--color-accent-primary)`

// You write:
context.t.var('themes.accent.primary')
// → 'var(--color-accent-primary)'

// Or look up the raw value:
context.t.value('tokens.spacing.4')
// → '1rem'

// Or get the CSS property name:
context.t.prop('tokens.colors.primary.500')
// → '--color-primary-500'
```

### Writing a Module

The button module as a complete example:

```typescript
// src/modules/components/button/index.ts

import type { StyForgeModule, BuildContext } from '../../../core/pipeline/types';

export const ButtonModule: StyForgeModule = {
  name: 'components/button',
  phase: 'components',

  isEnabled(config) {
    return config.components.enabled && config.components.button.enabled;
  },

  generate(ctx) {
    const { config, t } = ctx;
    const btn = config.components.button;

    const base = `
      .btn {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        font-family: inherit;
        font-weight: 500;
        border-radius: ${t.value('tokens.radius.md')};
        border: 1px solid transparent;
        cursor: pointer;
        text-decoration: none;
        transition: all ${t.value('tokens.transitions.fast')} ${t.value('tokens.transitions.easing.default')};
      }

      .btn:focus-visible {
        outline: 2px solid ${t.var('themes.accent.primary')};
        outline-offset: 2px;
      }

      .btn:disabled,
      .btn[aria-disabled="true"] {
        opacity: 0.5;
        cursor: not-allowed;
        pointer-events: none;
      }
    `;

    const sizes = Object.entries(btn.sizes).map(([name, size]) => `
      .btn--${name} {
        padding: ${t.resolveRef(size.padding)};
        font-size: ${t.resolveRef(size['font-size'])};
      }
    `).join('\n');

    const variants = btn.variants.map(variant => 
      generateVariant(variant, ctx)
    ).join('\n');

    return [base, sizes, variants].join('\n');
  }
};

function generateVariant(variant: string, ctx: BuildContext): string {
  const { t } = ctx;
  
  const variantMap = {
    primary: {
      bg: t.var('themes.accent.primary'),
      color: t.var('themes.accent.primary-contrast'),
      border: t.var('themes.accent.primary'),
      hoverBg: t.var('themes.accent.primary-hover'),
    },
    secondary: {
      bg: 'transparent',
      color: t.var('themes.accent.primary'),
      border: t.var('themes.accent.primary'),
      hoverBg: t.var('themes.colors.surface'),
    },
    ghost: {
      bg: 'transparent',
      color: t.var('themes.text.primary'),
      border: 'transparent',
      hoverBg: t.var('themes.colors.surface'),
    },
    danger: {
      bg: 'var(--color-danger-600)',
      color: '#ffffff',
      border: 'var(--color-danger-600)',
      hoverBg: 'var(--color-danger-700)',
    },
  };

  const v = variantMap[variant];
  if (!v) return '';

  return `
    .btn--${variant} {
      background-color: ${v.bg};
      color: ${v.color};
      border-color: ${v.border};
    }
    .btn--${variant}:hover:not(:disabled) {
      background-color: ${v.hoverBg};
    }
  `;
}
```

### Module Registration

```typescript
// src/core/pipeline/registry.ts

import { ButtonModule } from '../../modules/components/button';
import { CardModule } from '../../modules/components/card';
import { ResetModule } from '../../modules/reset';
import { BaseModule } from '../../modules/base';
// ...

export const DEFAULT_MODULES = [
  ResetModule,
  BaseModule,
  ButtonModule,
  CardModule,
  // Add new modules here. Zero other changes required.
];
```

---

## Stage 5: Output Stage

**Location:** `src/core/output/`

### Assembler

Takes the array of CSS strings from all modules and wraps them in the appropriate `@layer` declarations:

```typescript
// Input: array of { phase, css } tuples from modules
// Output: single assembled CSS string

function assemble(blocks: CSSBlock[], config: StyForgeConfig): string {
  const layerDeclaration = config.layers.enabled.join(', ');
  
  const grouped = groupByPhase(blocks);
  
  const sections = config.layers.enabled.map(layer => {
    const content = grouped[layer]?.join('\n') ?? '';
    if (!content.trim()) return '';
    return `@layer ${layer} {\n${indent(content)}\n}`;
  });

  return [
    generateHeader(config),
    `@layer ${layerDeclaration};`,
    '',
    sections.filter(Boolean).join('\n\n')
  ].join('\n');
}
```

### Output Header Comment

Every output file begins with a structured comment:

```css
/**
 * Generated by StyForge v1.0.0
 * Project: my-project
 * Built: 2025-10-15T14:23:01.000Z
 * Config: styforge.yaml
 *
 * Layers: tokens, reset, base, components
 * Themes: light (default), dark
 * Components: button, card, form, badge
 *
 * DO NOT EDIT — This file is auto-generated.
 * Edit styforge.yaml and run `styforge build` to regenerate.
 */
```

---

## Error Handling Philosophy

Errors in StyForge fall into two categories:

**Config Errors** — The user's YAML is invalid. These must be:
- Reported at the exact YAML location (file, line, column)
- Written in plain English with no jargon
- Accompanied by a suggested fix when possible
- Non-fatal as a group: report ALL config errors at once, not just the first

**Build Errors** — Something went wrong in the pipeline. These:
- Include a stack trace in verbose mode (`--verbose` flag)
- Report the module name that failed
- Are always fatal (the build stops)

```typescript
// Config error example
class ConfigError extends Error {
  constructor(
    public readonly path: string,      // "tokens.colors.primary.500"
    public readonly message: string,   // "must be a valid hex color"
    public readonly value: unknown,    // "#3G82F6"
    public readonly line?: number,
    public readonly column?: number,
  ) {
    super(message);
  }
}

// Formatted output:
// ✖ tokens.colors.primary.500 (line 14)
//   "#3G82F6" is not a valid hex color
//   Expected: #RRGGBB, #RGB, or #RRGGBBAA
//   Fix: Use a valid hex code, e.g. "#3B82F6"
```

---

## Testing Strategy

### Unit Tests
Every `src/core/` module has a corresponding test file. The token compiler and scale generators in particular need comprehensive coverage because errors there cascade to all output.

### Integration Tests
Full pipeline tests: a YAML config goes in, a CSS string comes out, we compare against a known-good snapshot.

```typescript
// tests/integration/build.test.ts
it('generates correct CSS from minimal config', async () => {
  const config = loadFixture('minimal.yaml');
  const output = await build(config);
  expect(output).toMatchSnapshot('minimal.css');
});
```

### Snapshot Tests
Snapshots are the primary correctness guarantee. When a change is intentional, snapshots are updated explicitly with `--update-snapshots`. Unintentional changes are caught by CI.

### Visual Regression (future)
For a v2 consideration: render the generated CSS in a headless browser and take screenshots of each component. Compare against approved screenshots.

---

## CLI Design

```
styforge <command> [options]

Commands:
  init [--preset <name>]    Create a new styforge.yaml in the current directory
  build [--config <path>]   Compile the config to CSS
  watch [--config <path>]   Watch for config changes and rebuild
  validate [--config <path>] Validate config without building
  list presets              List available built-in presets

Global Options:
  --config, -c   Path to config file (default: ./styforge.yaml)
  --verbose, -v  Enable verbose logging
  --quiet, -q    Suppress all output except errors
  --version      Print version number
  --help, -h     Show help

Examples:
  styforge init
  styforge init --preset editorial
  styforge build
  styforge build --config ./config/styles.yaml
  styforge watch
  styforge validate
```

---

## Dependency Philosophy

Dependencies are chosen conservatively. Every dependency is a liability (security surface, maintenance burden, breaking changes). Each must justify its presence.

| Package | Purpose | Justification |
|---------|---------|---------------|
| `yaml` | YAML parsing | The de-facto standard YAML library for Node.js. Stable, well-maintained. |
| `zod` | Schema validation | Best-in-class TypeScript validation. Generates types. The standard. |
| `chokidar` | File watching (watch mode) | The reliable cross-platform file watcher. Used by webpack, vite, jest. |
| `kleur` or `chalk` | Terminal colors | Error messages need color. One of these — the smaller one. |
| `commander` | CLI argument parsing | Industry standard. No reason to reinvent. |

**Not included:**
- PostCSS — We write CSS strings directly. No AST manipulation needed.
- Sass/Less — We output CSS variables. No preprocessor needed.
- Any CSS framework — We generate our own.
- Any bundler (webpack, vite, rollup) — We use TypeScript's own compiler (`tsc`) or `tsup` for the package build.

---

## TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

Notable choices:
- `exactOptionalPropertyTypes: true` — `{ x?: string }` means `x` is `string | undefined`, not `string`. Prevents bugs.
- `noUncheckedIndexedAccess: true` — Array/object access returns `T | undefined`. Prevents "cannot read property of undefined" bugs.
- `strict: true` — Non-negotiable.

---

*This document describes the intended implementation. It will be updated as implementation reveals gaps or better approaches.*
