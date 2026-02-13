# StyForge — Architecture Decision Records (ADR)

> Every significant architectural decision is recorded here with its context, the options considered, and the rationale for the choice made.  
> This is a living document. New decisions are appended. Old decisions are never deleted — only superseded.

---

## ADR Format

```
## ADR-{number} — {Title}
**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Superseded by ADR-{n}

### Context
What problem or question is this decision addressing?

### Options Considered
What alternatives were evaluated?

### Decision
What was decided and why?

### Consequences
What does this decision make easier? What does it make harder?
```

---

## ADR-001 — CSS Methodology: Tokens + Optional BEM Components

**Date:** Pre-implementation  
**Status:** Accepted

### Context
We need to decide the shape of every CSS class name in the output. This decision affects every project that uses StyForge and cannot easily be changed post-release.

### Options Considered
1. BEM-only class system
2. Utility-first classes
3. Semantic/component classes
4. CSS custom properties only (tokens only)
5. Hybrid: tokens always, BEM components opt-in

### Decision
**Option 5 — Hybrid.** The token layer (CSS custom properties) is always generated. The component CSS (BEM-structured) is opt-in per-component via config.

### Consequences
✓ Tokens are universally useful — any project benefits from a consistent token system  
✓ Components are optional — projects with their own component CSS don't get unwanted rules  
✓ BEM is readable and team-friendly  
✓ CSS `@layer` makes the entire output non-conflicting with user CSS  
✗ Two systems to learn (tokens + BEM), though both are industry standards  
✗ BEM is more verbose than utility-first in HTML

---

## ADR-002 — Theming: Semantic Tokens + CSS `[data-theme]` + `prefers-color-scheme`

**Date:** Pre-implementation  
**Status:** Accepted

### Context
Modern projects require dark mode support. We need to decide how theme switching works and how many files it produces.

### Options Considered
1. Single theme per build (no switching)
2. Multi-theme in one file via `[data-theme]` attribute
3. Separate CSS file per theme
4. CSS `@layer` cascade-based theming

### Decision
**Option 2 + 4 combined.** Semantic tokens (role-based, not value-based naming) live in `@layer tokens`. Themes override semantic tokens via `[data-theme="dark"]` and `@media (prefers-color-scheme: dark)` with the `root:not([data-theme="light"])` guard.

### Consequences
✓ One CSS file handles all themes  
✓ Zero JavaScript required to switch themes  
✓ OS preference respected by default  
✓ Explicit user choice overrides OS preference  
✓ Semantic token names (`--color-text-primary`) are self-documenting  
✗ Developers must use semantic tokens in their component CSS (can't use raw tokens) to get automatic dark mode  
✗ Slightly larger CSS file than single-theme output

---

## ADR-003 — YAML Schema: Progressive (Minimal Required, Full Available)

**Date:** Pre-implementation  
**Status:** Accepted

### Context
The power of StyForge is in the YAML config. But that power can become complexity. We need to balance expressiveness with approachability.

### Options Considered
1. Minimal YAML only (just tokens)
2. Full YAML (all options always visible)
3. Progressive YAML (minimal works, full available, defaults cover everything)

### Decision
**Option 3 — Progressive.** A 3-line config is valid. A 300-line config is also valid. Defaults cover everything that isn't specified. The schema is documented in YAML-REFERENCE.md.

### Consequences
✓ Low barrier to entry — productive in under 5 minutes  
✓ No artificial ceiling — power users get full control  
✓ Default config embeds best-practice decisions  
✗ Defaults must be thoughtfully chosen — they represent the "default" StyForge visual language  
✗ Users may not discover advanced features without reading the reference

---

## ADR-004 — Distribution: NPM Package + CLI (install globally or locally)

**Date:** Pre-implementation  
**Status:** Accepted

### Context
How developers get and use the tool has major implications for reproducibility, maintenance, and team usage.

### Options Considered
1. Global-only CLI
2. Per-project devDependency
3. NPM package that works as both global CLI and local devDependency
4. Monorepo only

### Decision
**Option 3.** The package exports a CLI binary (`bin`) that works both globally and locally. This is the standard pattern for serious Node.js tooling (eslint, prettier, typescript).

### Consequences
✓ Works everywhere without special configuration  
✓ Version pinnable per-project for reproducibility  
✓ `npx styforge@latest` works for one-off use without install  
✓ CI/CD environments work without global install  
✗ Minor: must be installed per-project for version pinning (not truly "install once")

---

## ADR-005 — Pipeline Architecture: Module-Based with Typed Interface

**Date:** Pre-implementation  
**Status:** Accepted

### Context
The build pipeline processes config through multiple stages and generates CSS from multiple concerns. The architecture determines how easy it is to add new components and how stable the core is.

### Options Considered
1. Monolithic compiler (one function, top to bottom)
2. Module-based pipeline with a defined module interface

### Decision
**Option 2.** Every CSS concern (reset, base, each component, utilities) is a module implementing a shared TypeScript interface. The pipeline discovers and executes modules. Adding a new component = creating a new module file. Zero changes to core.

### Consequences
✓ New components are isolated — can't break existing components  
✓ Modules are independently testable  
✓ Clear contribution path — one file per component  
✓ Core pipeline code is stable and rarely changes  
✗ More initial structure than a monolithic approach  
✗ Module registration step required for each new module

---

## ADR-006 — Output: Single File Default, Split Available

**Date:** Pre-implementation  
**Status:** Accepted

### Context
Should the output be one CSS file or multiple?

### Options Considered
1. Single file only
2. Split output only (tokens.css, components.css, etc.)
3. Configurable (single by default, split available)

### Decision
**Option 3.** Single file is the default because it requires the fewest decisions from the consumer. Split is available for projects that want surgical inclusion (e.g., only tokens, or tokens + reset but no components).

### Consequences
✓ Simple default (one `<link>` tag)  
✓ Power available when needed  
✓ Split mode enables "I just want your tokens" use case  
✗ Two output modes to test and maintain

---

## ADR-007 — Validation Library: Zod

**Date:** Pre-implementation  
**Status:** Accepted

### Context
The YAML config must be validated before the build runs. We need a validation approach that provides excellent error messages and TypeScript integration.

### Options Considered
1. Manual validation (if-statements)
2. JSON Schema + Ajv
3. Zod
4. Valibot

### Decision
**Zod.** It is the TypeScript-first validation standard. Schemas are TypeScript code (not JSON), making them easier to maintain. `z.infer<typeof Schema>` automatically generates TypeScript types, eliminating duplication. Error messages are clear. The ecosystem (including JSON Schema export via `zod-to-json-schema`) covers all our needs.

### Consequences
✓ One source of truth: Zod schema → TypeScript types + runtime validation + JSON Schema for IDE  
✓ Excellent error messages out of the box  
✓ TypeScript-native — refactoring is safe  
✗ Zod is a production dependency (small but present in the CLI's node_modules)  
✗ Zod v3 and v4 have different APIs — version must be pinned deliberately

---

## Open Architecture Questions

These are decisions that have not been made yet. They are tracked here until resolved.

| ID | Question | Options | Impact | Priority |
|----|---------|---------|--------|----------|
| OQ-001 | Should hover/focus/disabled states be included in component CSS by default, or opt-in? | Default on; default off; per-component config | DX, output size | High — before component implementation |
| OQ-002 | Should there be a user-facing plugin system? (i.e., `styforge.yaml` can reference external module packages) | Yes (adds complexity); No (internal modules only) | Architecture scope | High — before pipeline finalization |
| OQ-003 | What is the minimum Node.js version target? | Node 18 LTS; Node 20 LTS; Node 22 | Build constraints, ESM support | High — before package.json |
| OQ-004 | How are breaking YAML schema changes handled across versions? | Semver + migration guide; Versioned schema with `version:` key in YAML; Automated migration tool | Long-term maintainability | Medium — before v1.0 |
| OQ-005 | Color generation algorithm for auto-generated palettes? | OKLCH; HSL; culori library | Color quality, output accuracy | Medium — before token compiler |
| OQ-006 | Should `styforge watch` support watching multiple config files? (monorepo use case) | Yes; No | Monorepo DX | Low — post-v1 |
| OQ-007 | Should presets be built-in (bundled) or fetchable from a registry? | Built-in only; Registry with built-in fallbacks | Distribution, ecosystem growth | Low — post-v1 |
| OQ-008 | ESM or CJS output for the package? | ESM only; CJS only; Dual (ESM + CJS) | Compatibility | Medium — before first release |

---

## Decision Log Summary

| ADR | Decision | Status |
|-----|---------|--------|
| ADR-001 | CSS Methodology: Tokens + Optional BEM | ✅ Accepted |
| ADR-002 | Theming: `[data-theme]` + `prefers-color-scheme` | ✅ Accepted |
| ADR-003 | YAML: Progressive schema | ✅ Accepted |
| ADR-004 | Distribution: NPM package + CLI | ✅ Accepted |
| ADR-005 | Pipeline: Module-based architecture | ✅ Accepted |
| ADR-006 | Output: Single file default, split available | ✅ Accepted |
| ADR-007 | Validation: Zod | ✅ Accepted |
| OQ-001 | Hover/focus/disabled states | ⏳ Open |
| OQ-002 | Plugin system | ⏳ Open |
| OQ-003 | Node.js minimum version | ⏳ Open |
| OQ-004 | Schema versioning strategy | ⏳ Open |
| OQ-005 | Color generation algorithm | ⏳ Open |
| OQ-006 | Multi-config watch mode | ⏳ Open |
| OQ-007 | Preset distribution | ⏳ Open |
| OQ-008 | ESM/CJS output format | ⏳ Open |
