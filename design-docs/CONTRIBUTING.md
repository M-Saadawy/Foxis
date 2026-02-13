# StyForge — Contributing Guide

> How to add new components, extend the pipeline, and contribute to StyForge.

---

## Adding a New Component Module

This is the most common contribution. Here is the complete process for adding a new component.

### Step 1 — Add the YAML schema

In `src/core/config/schema.ts`, add the Zod schema for your component's config:

```typescript
const TooltipConfigSchema = z.object({
  enabled: z.boolean().default(true),
  radius: z.string().default('{tokens.radius.sm}'),
  max-width: z.string().default('200px'),
  arrow: z.boolean().default(true),
}).default({});

// Add to ComponentsConfigSchema:
export const ComponentsConfigSchema = z.object({
  // ... existing components
  tooltip: TooltipConfigSchema,
});
```

### Step 2 — Add the TypeScript type

Types are inferred from Zod schemas automatically. You just need to re-export if needed:

```typescript
// src/core/config/types.ts
export type TooltipConfig = z.infer<typeof TooltipConfigSchema>;
```

### Step 3 — Add the default config

In `src/core/config/defaults.ts`:

```typescript
export const DEFAULT_CONFIG: DeepPartial<StyForgeConfig> = {
  // ... existing defaults
  components: {
    // ... existing component defaults
    tooltip: {
      enabled: true,
      radius: '{tokens.radius.sm}',
      'max-width': '200px',
      arrow: true,
    }
  }
};
```

### Step 4 — Create the module

Create `src/modules/components/tooltip/index.ts`:

```typescript
import type { StyForgeModule, BuildContext } from '../../../core/pipeline/types';

export const TooltipModule: StyForgeModule = {
  name: 'components/tooltip',
  phase: 'components',

  isEnabled(config) {
    return (
      config.components.enabled !== false &&
      config.components.tooltip?.enabled !== false
    );
  },

  generate(ctx) {
    const { config, t } = ctx;
    const tooltip = config.components.tooltip;

    return `
      /* ── Tooltip ── */
      .tooltip-wrapper {
        position: relative;
        display: inline-flex;
      }

      .tooltip {
        position: absolute;
        bottom: calc(100% + 8px);
        left: 50%;
        transform: translateX(-50%);
        max-width: ${t.resolveRef(tooltip['max-width'])};
        padding: ${t.value('tokens.spacing.1')} ${t.value('tokens.spacing.2')};
        background-color: ${t.var('themes.colors.text-primary')};
        color: ${t.var('themes.colors.background')};
        font-size: ${t.value('tokens.typography.sizes.xs')};
        line-height: ${t.value('tokens.typography.line-heights.tight')};
        border-radius: ${t.resolveRef(tooltip.radius)};
        white-space: nowrap;
        pointer-events: none;
        opacity: 0;
        transition: opacity ${t.value('tokens.transitions.duration.fast')} ${t.value('tokens.transitions.easing.default')};
        z-index: 50;
      }

      .tooltip-wrapper:hover .tooltip,
      .tooltip-wrapper:focus-within .tooltip {
        opacity: 1;
      }

      ${tooltip.arrow ? `
      .tooltip::after {
        content: '';
        position: absolute;
        top: 100%;
        left: 50%;
        transform: translateX(-50%);
        border: 5px solid transparent;
        border-top-color: ${t.var('themes.colors.text-primary')};
      }
      ` : ''}
    `;
  }
};
```

### Step 5 — Register the module

In `src/core/pipeline/registry.ts`:

```typescript
import { TooltipModule } from '../../modules/components/tooltip';

export const DEFAULT_MODULES = [
  // ... existing modules
  TooltipModule,  // ← Add here
];
```

### Step 6 — Write tests

Create `tests/unit/modules/tooltip.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { TooltipModule } from '../../../src/modules/components/tooltip';
import { createMockContext } from '../../helpers/context';

describe('TooltipModule', () => {
  it('is enabled by default', () => {
    const ctx = createMockContext();
    expect(TooltipModule.isEnabled(ctx.config)).toBe(true);
  });

  it('is disabled when components.enabled is false', () => {
    const ctx = createMockContext({ components: { enabled: false } });
    expect(TooltipModule.isEnabled(ctx.config)).toBe(false);
  });

  it('generates .tooltip class', () => {
    const ctx = createMockContext();
    const css = TooltipModule.generate(ctx);
    expect(css).toContain('.tooltip {');
  });

  it('omits arrow CSS when arrow is false', () => {
    const ctx = createMockContext({ components: { tooltip: { arrow: false } } });
    const css = TooltipModule.generate(ctx);
    expect(css).not.toContain('.tooltip::after');
  });
});
```

Create `tests/integration/tooltip.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { build } from '../../../src';

it('tooltip CSS matches snapshot', async () => {
  const css = await build({
    components: { tooltip: { enabled: true } }
  });
  expect(css).toMatchSnapshot();
});
```

### Step 7 — Update YAML Reference

Add your component to `docs/YAML-REFERENCE.md` under the `components` section, following the existing format.

### Step 8 — Update the JSON Schema

Run `npm run generate-schema` to regenerate `schema/styforge.schema.json` from the Zod schemas. This updates IDE autocompletion.

---

## The Module Contract

Every module must satisfy this contract:

```typescript
interface StyForgeModule {
  name: string;       // Format: "phase/component-name". Unique. Kebab-case.
  phase: Phase;       // 'tokens' | 'reset' | 'base' | 'components' | 'utilities'
  isEnabled(config: StyForgeConfig): boolean;  // Pure function. No side effects.
  generate(context: BuildContext): string;      // Returns CSS string. No side effects.
}
```

### `isEnabled` Rules
- Must be a pure function with no side effects
- Must handle missing/undefined config gracefully (use optional chaining)
- Must check `config.components.enabled` (the master switch) before checking component-specific flags
- Should return `false` (not throw) for any invalid config — validation happens earlier

### `generate` Rules
- Must return valid CSS as a string
- Must not include `@layer` wrapper — the pipeline adds that
- Should use `context.t` (TokenAccessor) for all token values — not hardcoded values
- Should use `/* ── ComponentName ── */` header comment before each component's CSS
- All generated CSS must be responsive to the config — no hardcoded values that belong in config
- Must handle `config.output.prefix` — use `prefix(className)` helper from context

---

## Code Style

### TypeScript
- Strict mode. No `any`. No non-null assertions without a comment explaining why.
- Prefer `const` and functional patterns. Avoid mutation.
- Explicit return types on all exported functions.
- All public interfaces documented with JSDoc.

### CSS in TypeScript
- Use template literals for CSS strings. One property per line.
- Indent generated CSS with two spaces.
- Group properties: positioning → box model → typography → visual → transitions.
- Comment non-obvious properties.

### Naming
- Files: `kebab-case.ts`
- Classes (TypeScript): `PascalCase`
- Interfaces: `PascalCase` with `I` prefix only for disambiguation
- Functions: `camelCase`
- Constants: `UPPER_SNAKE_CASE` for true constants, `camelCase` for module exports
- CSS classes in output: `.kebab-case`, BEM with `__` and `--`

---

## Testing Requirements

All contributions must include:

| Type | Requirement |
|------|------------|
| Unit test | For `isEnabled` (all cases) |
| Unit test | For `generate` (happy path + config variants) |
| Snapshot test | Full build output for new component |
| YAML schema test | New config keys validate correctly |

Run tests:
```bash
npm test              # All tests
npm run test:unit     # Unit only
npm run test:snapshot # Snapshot tests only
npm run test:update   # Update snapshots (use deliberately)
```

---

## Pull Request Process

1. Fork the repository and create a branch: `feature/component-tooltip`
2. Make your changes following this guide
3. Run `npm test` — all tests must pass
4. Run `npm run lint` — no linting errors
5. Run `npm run type-check` — no type errors
6. Update documentation (YAML-REFERENCE.md, at minimum)
7. Regenerate the JSON schema: `npm run generate-schema`
8. Open a PR with:
   - A description of what the component does
   - The YAML config snippet needed to use it
   - A sample of the CSS output

---

## Common Mistakes

**Using hardcoded values in modules**
```typescript
// ❌ Wrong
`.btn { padding: 8px 16px; }`

// ✅ Right
`.btn { padding: ${t.value('tokens.spacing.2')} ${t.value('tokens.spacing.4')}; }`
```

**Not checking the master switch**
```typescript
// ❌ Wrong
isEnabled(config) {
  return config.components.badge?.enabled !== false;
}

// ✅ Right
isEnabled(config) {
  return (
    config.components.enabled !== false &&
    config.components.badge?.enabled !== false
  );
}
```

**Generating CSS without theme awareness**
```typescript
// ❌ Wrong — hardcoded color, won't change with dark mode
`.card { background: #FFFFFF; }`

// ✅ Right — uses semantic theme token, changes with theme
`.card { background: ${t.var('themes.colors.surface-raised')}; }`
```

**Adding `@layer` wrapper inside a module**
```typescript
// ❌ Wrong — the pipeline adds @layer, don't double-wrap
`@layer components {
  .btn { }
}`

// ✅ Right — just return the CSS content
`.btn { }`
```
