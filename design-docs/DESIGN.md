# StyForge — Master Design Document

> **Status:** Pre-implementation · Design Phase  
> **Version:** 0.1.0-design  
> **Purpose:** This document exists to explore the problem space, map every architectural road, weigh the tradeoffs, and arrive at a well-reasoned set of decisions before a single line of production code is written.

---

## Table of Contents

1. [The Problem We're Solving](#1-the-problem-were-solving)
2. [Vision & Guiding Philosophy](#2-vision--guiding-philosophy)
3. [What This Is Not](#3-what-this-is-not)
4. [Core Concepts & Vocabulary](#4-core-concepts--vocabulary)
5. [Road A — CSS Methodology](#5-road-a--css-methodology)
6. [Road B — Theming Strategy](#6-road-b--theming-strategy)
7. [Road C — YAML Granularity](#7-road-c--yaml-granularity)
8. [Road D — Project Distribution Model](#8-road-d--project-distribution-model)
9. [Road E — Build Pipeline Architecture](#9-road-e--build-pipeline-architecture)
10. [Road F — Output Format Strategy](#10-road-f--output-format-strategy)
11. [Recommended Architecture Decision Record (ADR)](#11-recommended-architecture-decision-record-adr)
12. [Proposed Directory Structure](#12-proposed-directory-structure)
13. [YAML Schema Design](#13-yaml-schema-design)
14. [TypeScript Pipeline Design](#14-typescript-pipeline-design)
15. [Developer Experience (DX) Goals](#15-developer-experience-dx-goals)
16. [Open Questions](#16-open-questions)

---

## 1. The Problem We're Solving

Every project a developer starts from scratch carries a silent tax: the time and mental overhead of re-establishing the same foundational CSS decisions. Colors, spacing scales, typography stacks, resets, component patterns — these are reimplemented, copy-pasted, or forgotten project after project.

The existing solutions either go too far (Tailwind, Bootstrap — they impose a full philosophy and visual system) or don't go far enough (a `reset.css` file does nothing for design tokens or components).

**The gap we're filling:**

> A tool that lets a developer describe *what they want their CSS foundation to look like* in a simple config file, and get back a clean, vanilla CSS file with zero runtime dependencies — completely usable in any project regardless of framework or build tooling.

The config file is the product. The CSS output is just the artifact.

---

## 2. Vision & Guiding Philosophy

### The Three Laws

**Law 1 — Zero runtime footprint.**
The output is plain CSS. No JavaScript. No class-name generation at runtime. No PostCSS required in the consuming project. A developer should be able to drop the output file into a plain HTML project from 2003 and it would work.

**Law 2 — The YAML file is the source of truth.**
Every design decision lives in the YAML config. The tool is a compiler: YAML in, CSS out. Nothing should require editing template files or touching source code to achieve customization.

**Law 3 — Opinionated defaults, total override-ability.**
Ship with sensible defaults so a developer can run the tool with zero config and get something usable. But expose every knob. Nothing is hardcoded in a way that can't be overridden.

### The Mental Model

Think of this tool the way you think of a compiler:
- **Source language:** YAML (the config you write)
- **Intermediate representation:** An in-memory design token tree (TypeScript)
- **Target language:** Plain CSS (the file you ship)

The tool is not a CSS framework. It is a CSS *factory*. The factory has a spec sheet (YAML). The spec sheet produces the product (CSS).

---

## 3. What This Is Not

Clarity about what this tool is *not* is as important as what it is.

| It is NOT | Why this matters |
|-----------|-----------------|
| A CSS framework | Frameworks impose a visual system. This tool generates *your* visual system. |
| A Tailwind replacement | Tailwind is utility-first and requires a build step in every project. This tool outputs framework-agnostic CSS. |
| A design system | Design systems include documentation, governance, and often JavaScript components. This is the CSS layer only. |
| A preprocessor (like Sass) | Sass still requires a build step in the consuming project. This tool's output requires nothing. |
| A component library | It can generate component *CSS*, but not markup or JavaScript behavior. |
| A one-size-fits-all solution | Different projects have different needs. The tool serves as a personal scaffold, not a universal standard. |

---

## 4. Core Concepts & Vocabulary

These terms are used consistently throughout all documentation.

| Term | Definition |
|------|-----------|
| **Config** | The YAML file that describes what CSS to generate. The user's primary interface. |
| **Token** | A named design value (a color, a spacing unit, a font size). Tokens become CSS custom properties. |
| **Scale** | A set of related tokens (e.g. a spacing scale: `4px, 8px, 16px, 32px, 64px`). |
| **Theme** | A complete named set of tokens. Multiple themes can coexist in one output file. |
| **Layer** | A CSS `@layer` block. Layers control cascade precedence and provide a clean separation of concerns. |
| **Module** | A section of the build pipeline responsible for one concern (e.g., the Reset module, the Typography module). |
| **Artifact** | The final compiled CSS file(s) output by the build. |
| **Preset** | A pre-authored YAML config that captures a complete design style (e.g., "minimal", "editorial", "brutalist"). |

---

## 5. Road A — CSS Methodology

This is the single most consequential architectural decision. It determines the shape of every CSS class name in the output file, and the developer experience in every project that consumes it.

### Option A1 — BEM (Block__Element--Modifier)

**What it looks like:**
```css
.btn { }
.btn--primary { }
.btn--large { }
.btn__icon { }
.btn__icon--left { }
```

**Pros:**
- Extremely readable. Every class name describes its position in the component hierarchy.
- Zero specificity wars. All classes are single-level.
- Scales beautifully for teams. New developers immediately understand the class hierarchy.
- No need to learn a new system. BEM is 15 years old and universally understood.
- Works identically in any HTML/template system.

**Cons:**
- Verbose class names. `card__header--highlighted` is a lot to type.
- Requires developer discipline to stick to BEM conventions. Easy to drift.
- Can feel overly ceremonial for small projects.
- Generates larger HTML files (class name verbosity).

**Best for:** Teams, component-driven projects, long-lived codebases, projects where HTML is authored by multiple developers.

---

### Option A2 — Utility-First

**What it looks like:**
```css
.flex { display: flex; }
.items-center { align-items: center; }
.gap-4 { gap: 1rem; }
.text-lg { font-size: 1.125rem; }
.bg-primary { background-color: var(--color-primary); }
```

**Pros:**
- Maximum flexibility. Any combination of properties in HTML without writing new CSS.
- No naming decisions. Classes describe what they do, not what they are.
- Tiny component CSS surface area. Most styling happens at the HTML layer.
- Very fast to prototype.

**Cons:**
- HTML becomes visually noisy. `class="flex items-center gap-4 px-6 py-3 bg-primary text-white rounded-lg"` is hard to read.
- Design intent is invisible in the CSS. You can't look at a class and know what component it styles.
- Reinvents Tailwind, but worse. Tailwind is already the definitive utility-first solution. Building a custom utility system invites constant comparison and will always lose.
- Harder to maintain semantic consistency across a project.

**Best for:** Rapid prototyping, single-developer projects, projects where Tailwind is already available.

**Verdict on A2:** Building a custom utility system is reinventing a solved problem and doing it less completely. Unless we have a strong reason, this road leads to "a worse Tailwind."

---

### Option A3 — Semantic / Component Classes

**What it looks like:**
```css
.card { }
.card-header { }
.button { }
.button.primary { }
.button.large { }
.navigation { }
.navigation a { }
```

**Pros:**
- Simple, readable, approachable. Easy for beginners.
- HTML stays clean. One or two meaningful class names per element.
- Mirrors how most developers naturally think about HTML.

**Cons:**
- Specificity escalates quickly. `.card .button.primary:hover` becomes a nightmare.
- Naming collisions in large projects. Without a convention, `header` could mean 10 different things.
- Hard to compose. Semantic classes describe things, not properties, making reuse difficult.

**Best for:** Small projects, solo developers, simple marketing sites.

**Verdict on A3:** Too brittle at scale. Good for simple projects, but the tool claims "enterprise-grade commitment" which means this doesn't hold up.

---

### Option A4 — CSS Custom Properties / Design Tokens Only (No class system)

**What it looks like:**
```css
:root {
  --color-primary: #2563EB;
  --spacing-md: 1rem;
  --font-size-lg: 1.125rem;
  --radius-default: 0.375rem;
}
```

The tool *only* generates tokens. No component classes at all.

**Pros:**
- Maximum flexibility. The developer builds their own classes on top of tokens.
- Zero opinion on class naming, methodology, or structure.
- Tiny output. Just variables.
- Compatible with *any* CSS methodology. Use it alongside BEM, Tailwind, CSS Modules, anything.
- True separation: the token layer is universal, the class layer is project-specific.

**Cons:**
- Provides the least immediate value. A developer still has to write all their component CSS.
- Without classes, the "skeleton" aspect is minimal.
- The less we generate, the less differentiated this tool is from just maintaining a variables file.

**Best for:** Projects already using a CSS methodology, teams wanting a token standard without methodology lock-in.

---

### Option A5 — Hybrid: Tokens + Optional Component Layer

**What it looks like:**
```css
/* Layer 1: Always generated — Design tokens */
:root {
  --color-primary: #2563EB;
  --spacing-md: 1rem;
}

/* Layer 2: Opt-in — Component classes (BEM-structured) */
@layer components {
  .btn { padding: var(--spacing-sm) var(--spacing-md); }
  .btn--primary { background: var(--color-primary); }
}
```

The YAML config controls which layers are generated.

**Pros:**
- Best of all worlds. Tokens are always present. Components are opt-in.
- CSS `@layer` gives consuming projects a clean override mechanism.
- Composable. A developer can use just tokens, or tokens + components, or everything.
- Methodologically honest: tokens are universal, components are opinionated.

**Cons:**
- More complex to implement. Multiple generation modes.
- The BEM component layer still requires a methodology decision.
- Risk of generating unused CSS if developers enable everything.

**Best for:** The primary use case — a personal skeleton that works across projects of varying complexity.

**Verdict on A5: This is the recommended road.** The token layer is the non-negotiable core. The component layer is opt-in and BEM-structured. CSS `@layer` makes it all composable and overrideable.

---

## 6. Road B — Theming Strategy

Once we have tokens, we need to decide how multiple themes (e.g., light and dark mode, or completely different visual styles) are handled.

### Option B1 — Single Theme Per Build

One build = one YAML = one CSS file with one set of values. No theme switching.

**Pros:** Simplest output. Smallest CSS file. No complexity. No decisions at runtime.  
**Cons:** Dark mode requires a separate build. No CSS-layer theme switching. Two separate CSS files to maintain per project.

---

### Option B2 — Multi-Theme in One File (`:root` overrides)

```css
:root {
  --color-bg: #ffffff;
  --color-text: #111111;
}

[data-theme="dark"] {
  --color-bg: #111111;
  --color-text: #ffffff;
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --color-bg: #111111;
    --color-text: #ffffff;
  }
}
```

**Pros:** One CSS file handles everything. Theme switching is zero-JS (just a `data-theme` attribute). Respects `prefers-color-scheme`. This is the modern standard pattern.  
**Cons:** Slightly larger CSS file. Requires semantic token naming (tokens represent *roles*, not values: `--color-bg` not `--color-white`).

---

### Option B3 — Separate CSS File Per Theme

```
output/
  base.css        ← shared reset, typography, components
  theme-light.css ← light theme tokens
  theme-dark.css  ← dark theme tokens
  theme-brand.css ← a custom brand theme
```

**Pros:** Very clean separation. Load only what you need. Theoretically the smallest per-theme payload.  
**Cons:** Multiple `<link>` tags. Switching themes requires JavaScript to swap stylesheets. More complex DX. The performance gain is negligible vs. B2.

---

### Option B4 — CSS `@layer` with Cascade-Based Theming

```css
@layer tokens, components, utilities, overrides;

@layer tokens {
  :root { --color-bg: #fff; }
}
```

**Pros:** Maximum cascade control. Consuming project can override any layer without specificity fights.  
**Cons:** `@layer` support, while now near-universal, means older browsers (pre-2022) get no cascade control.

---

### Recommendation: B2 + B4 Combined

Use semantic tokens (role-based naming) inside `@layer tokens`. Support multi-theme via `:root` / `[data-theme]` / `prefers-color-scheme`. This is the modern, correct approach and is now fully supported by all browsers in active use.

---

## 7. Road C — YAML Granularity

How much of the CSS system does the YAML control? Three possible philosophies:

### Option C1 — Minimal YAML (Tokens Only)

```yaml
colors:
  primary: "#2563EB"
  background: "#FFFFFF"
spacing:
  base: 4
  scale: [1, 2, 3, 4, 6, 8, 12, 16, 24]
typography:
  font-sans: "Inter, system-ui, sans-serif"
  size-base: 16
```

**Pros:** Simple to learn. Small config file. Focused.  
**Cons:** Limited power. Doesn't control component generation, reset behavior, layer structure, or output format.

---

### Option C2 — Full Control YAML

```yaml
meta:
  name: "my-project"
  version: "1.0.0"
  output: "./dist/styles.css"

layers:
  enabled: [tokens, reset, base, components, utilities]
  order: [tokens, reset, base, components, utilities]

tokens:
  colors:
    primary:
      50: "#EFF6FF"
      500: "#3B82F6"
      900: "#1E3A8A"
  spacing:
    scale: fibonacci
    base: 4
  typography:
    families:
      sans: { stack: "Inter, system-ui, sans-serif", weights: [400, 600, 700] }
      mono: { stack: "JetBrains Mono, monospace", weights: [400] }
    sizes:
      scale: "major-third"
      base: 16

themes:
  default: light
  light:
    colors:
      background: "{tokens.colors.white}"
      surface: "{tokens.colors.gray.50}"
      text: "{tokens.colors.gray.900}"
  dark:
    colors:
      background: "{tokens.colors.gray.950}"
      surface: "{tokens.colors.gray.900}"
      text: "{tokens.colors.gray.50}"

components:
  button:
    enabled: true
    variants: [primary, secondary, ghost, danger]
    sizes: [sm, md, lg]
    radius: "{tokens.radius.md}"
  card:
    enabled: true
    padding: "{tokens.spacing.6}"

reset:
  type: modern       # modern | normalize | opinionated | none
  box-sizing: border-box

output:
  format: single     # single | split
  minify: false
  source-map: false
  prefix: ""         # optional class prefix e.g. "sf-" → .sf-btn
```

**Pros:** Maximum power and flexibility. Every decision is documented in one file. Version-controlled. Self-describing. Enterprise-grade.  
**Cons:** Steeper learning curve. A developer opening this for the first time might be overwhelmed.

---

### Option C3 — Progressive YAML (Minimal Required, Full Optional)

This is C1 and C2 combined via schema defaults: the minimal YAML of C1 *works*, and the full YAML of C2 is *available* but never required.

```yaml
# This is a valid, complete config:
colors:
  primary: "#2563EB"

# And so is this:
meta:
  name: "full-project"
layers:
  enabled: [tokens, reset, components]
tokens:
  colors:
    primary:
      500: "#2563EB"
# ... etc
```

**Recommendation: C3.** Defaults should cover 90% of use cases. Advanced keys unlock the other 10%. A developer should be productive in under 5 minutes with a 5-line config.

---

## 8. Road D — Project Distribution Model

How does a developer get this tool and use it across projects?

### Option D1 — Global CLI Tool

```bash
npm install -g styforge
styforge init          # creates styforge.yaml in current dir
styforge build         # compiles → dist/styles.css
styforge watch         # watches config, rebuilds on change
```

**Pros:**
- Install once, use anywhere. True "skeleton" behavior across projects.
- Projects themselves have zero dev dependencies related to this tool.
- Clean separation: the tool is infrastructure, not project code.
- Easiest to update: `npm update -g styforge` updates everywhere.

**Cons:**
- Global installs are increasingly discouraged in modern Node.js practice (`npx` is preferred).
- Version mismatches across machines in a team. Developer A uses v1.0, Developer B uses v2.0.
- CI/CD environments need the tool installed separately.

---

### Option D2 — Per-Project DevDependency

```bash
npm install --save-dev styforge
npx styforge build
```

**Pros:**
- Version is pinned in `package.json`. Reproducible across team and CI.
- Standard, expected modern Node.js practice.
- `package.json` scripts make it first-class:
  ```json
  {
    "scripts": {
      "css:build": "styforge build",
      "css:watch": "styforge watch"
    }
  }
  ```

**Cons:**
- Must be installed in every new project. Not "install once."
- Slightly increases the `node_modules` footprint of every project (though this is a minor concern — `node_modules` are already enormous).

---

### Option D3 — NPM Package + CLI (Hybrid)

The package serves dual purpose: it can be installed globally for CLI use, or locally as a dev dependency. Same package, same binary, both modes work.

This is what tools like `prettier`, `eslint`, and `typescript` do. It is the industry standard for serious developer tooling.

```bash
# Global use
npm install -g styforge

# Or per-project
npm install --save-dev styforge

# Or no install at all
npx styforge@latest build
```

**Recommendation: D3.** This is simply the correct modern approach. It respects how the Node.js ecosystem works and how developers expect CLI tools to behave.

---

### Option D4 — Monorepo with Shared Package

For teams maintaining multiple projects under a single repo (Nx, Turborepo, pnpm workspaces):

```
monorepo/
  packages/
    styforge-config/      ← shared YAML configs and presets
  apps/
    project-a/
    project-b/
```

**Pros:** One config, many projects. Central updates cascade.  
**Cons:** This is an organizational pattern built on top of D3, not an alternative to it. If someone is in a monorepo, D3 still works — they just share the YAML config.

**Verdict:** D4 is a usage pattern, not a distribution model. It's automatically supported by D3.

---

## 9. Road E — Build Pipeline Architecture

The TypeScript pipeline that reads YAML and outputs CSS. This is the internal engine.

### Key Pipeline Stages

Regardless of which architectural choices we make elsewhere, the pipeline always has these stages:

```
[1] Config Resolution
    └─ Load styforge.yaml
    └─ Merge with defaults
    └─ Validate against schema
    └─ Resolve token references (e.g. "{tokens.colors.primary.500}")

[2] Token Compilation
    └─ Expand scales (spacing, type, color palettes)
    └─ Compute derived tokens (color shades, responsive sizes)
    └─ Build the token tree (in-memory representation)

[3] Theme Resolution
    └─ Apply light/dark/custom theme overrides
    └─ Generate semantic token mappings

[4] Module Execution
    └─ Reset module
    └─ Base/typography module
    └─ Component modules (button, card, form, etc.)
    └─ Utility module (if enabled)

[5] CSS Assembly
    └─ Wrap in @layer blocks
    └─ Apply prefix (if configured)
    └─ Assemble final CSS string

[6] Output
    └─ Minify (if configured)
    └─ Write to disk
    └─ Source map (if configured)
```

### Option E1 — Monolithic Compiler

One large function that processes everything top to bottom. Simple to build, hard to extend.

**Verdict:** Not suitable for an enterprise-grade tool with a commitment to extensibility.

### Option E2 — Module-Based Pipeline with Plugin System

Each stage in the pipeline is a TypeScript module with a defined interface. The pipeline is a sequence of module executions. New modules can be added without touching existing code.

```typescript
interface StyForgeModule {
  name: string;
  phase: 'tokens' | 'theme' | 'component' | 'utility';
  generate(context: BuildContext): CSSBlock[];
}
```

**Pros:** Clean separation of concerns. New components = new modules. Core is stable, extensible surface is large. Testable in isolation. This is how serious build tools work (PostCSS plugins, Webpack loaders, etc.).

**Verdict: E2 is the only acceptable architecture for a tool with enterprise ambitions.**

---

## 10. Road F — Output Format Strategy

What does the final CSS file look like structurally?

### Option F1 — Single File, Everything

```
dist/
  styles.css     ← everything: reset + tokens + components
```

**Pros:** One `<link>` tag. Simple. No decisions.  
**Cons:** Can't use just the tokens in a project that has its own component CSS.

### Option F2 — Split Output

```
dist/
  tokens.css     ← just CSS custom properties
  reset.css      ← just the reset
  components.css ← just the components
  index.css      ← @import of all of the above
```

**Pros:** Surgical inclusion. A developer can `<link>` just `tokens.css` if that's all they need. Excellent for projects that already have component CSS and just want the token system.

**Cons:** Multiple files. More complexity in the output configuration.

### Option F3 — Configurable (Single or Split)

The YAML config controls this:

```yaml
output:
  format: single    # → dist/styles.css
  # format: split  # → dist/tokens.css, dist/reset.css, etc.
```

**Recommendation: F3 with single as the default.** Single file is the 90% case. Split is available for power users.

---

## 11. Recommended Architecture Decision Record (ADR)

Based on the analysis above, these are the recommended decisions:

| Decision | Choice | Rationale |
|----------|--------|-----------|
| CSS Methodology | **A5: Tokens + Optional BEM Components** | Maximally composable. Tokens are universal. Components are opt-in. |
| Theming | **B2 + B4: Semantic tokens in `@layer`, multi-theme via `:root`/`[data-theme]`** | Modern standard. Zero-JS theme switching. Full browser support. |
| YAML Granularity | **C3: Progressive YAML** | 5-line config works. Full config available. No artificial ceiling. |
| Distribution | **D3: NPM Package + CLI** | Industry standard for developer tooling. Flexible. Reproducible. |
| Pipeline Architecture | **E2: Module-based with plugin system** | The only acceptable choice for a tool with enterprise ambitions. |
| Output Format | **F3: Configurable, single by default** | Simple default. Power available. |

---

## 12. Proposed Directory Structure

```
styforge/
│
├── packages/
│   └── styforge/                    ← The main package
│       ├── src/
│       │   ├── cli/
│       │   │   ├── index.ts         ← CLI entry point (commands: init, build, watch, validate)
│       │   │   ├── commands/
│       │   │   │   ├── build.ts
│       │   │   │   ├── init.ts
│       │   │   │   ├── watch.ts
│       │   │   │   └── validate.ts
│       │   │   └── logger.ts        ← Consistent terminal output formatting
│       │   │
│       │   ├── core/
│       │   │   ├── config/
│       │   │   │   ├── loader.ts    ← Reads and parses YAML
│       │   │   │   ├── validator.ts ← Zod schema validation
│       │   │   │   ├── resolver.ts  ← Merges config with defaults, resolves token refs
│       │   │   │   └── defaults.ts  ← Full default config tree
│       │   │   │
│       │   │   ├── tokens/
│       │   │   │   ├── compiler.ts  ← Builds the token tree
│       │   │   │   ├── scales.ts    ← Scale generation (spacing, type, color)
│       │   │   │   ├── themes.ts    ← Theme resolution and semantic mapping
│       │   │   │   └── references.ts ← Token reference resolution (e.g. "{tokens.x}")
│       │   │   │
│       │   │   ├── pipeline/
│       │   │   │   ├── runner.ts    ← Executes modules in order
│       │   │   │   ├── context.ts   ← BuildContext type: shared state across modules
│       │   │   │   └── types.ts     ← StyForgeModule interface and related types
│       │   │   │
│       │   │   └── output/
│       │   │       ├── assembler.ts ← Assembles CSS blocks into final string
│       │   │       ├── minifier.ts  ← Optional CSS minification
│       │   │       └── writer.ts    ← Writes file(s) to disk
│       │   │
│       │   ├── modules/             ← Each module = one concern
│       │   │   ├── reset/
│       │   │   │   ├── index.ts
│       │   │   │   ├── modern.css.ts    ← Modern reset template
│       │   │   │   ├── normalize.css.ts ← Normalize.css-based template
│       │   │   │   └── opinionated.css.ts
│       │   │   │
│       │   │   ├── base/
│       │   │   │   ├── index.ts
│       │   │   │   ├── typography.ts
│       │   │   │   └── body.ts
│       │   │   │
│       │   │   ├── components/
│       │   │   │   ├── button/
│       │   │   │   │   ├── index.ts
│       │   │   │   │   └── variants.ts
│       │   │   │   ├── card/
│       │   │   │   ├── form/
│       │   │   │   │   ├── input.ts
│       │   │   │   │   ├── select.ts
│       │   │   │   │   ├── checkbox.ts
│       │   │   │   │   └── label.ts
│       │   │   │   ├── badge/
│       │   │   │   ├── alert/
│       │   │   │   └── nav/
│       │   │   │
│       │   │   └── utilities/       ← Optional utility classes (opt-in)
│       │   │       ├── layout.ts
│       │   │       ├── spacing.ts
│       │   │       └── text.ts
│       │   │
│       │   ├── presets/             ← Pre-built YAML configs (starter themes)
│       │   │   ├── minimal.yaml
│       │   │   ├── editorial.yaml
│       │   │   └── dashboard.yaml
│       │   │
│       │   └── index.ts             ← Public API (for programmatic use)
│       │
│       ├── tests/
│       │   ├── unit/
│       │   │   ├── tokens/
│       │   │   ├── modules/
│       │   │   └── config/
│       │   ├── integration/
│       │   │   └── build.test.ts    ← Full pipeline: YAML in → CSS out
│       │   └── snapshots/           ← CSS snapshot tests
│       │
│       ├── schema/
│       │   └── styforge.schema.json ← JSON Schema for IDE autocompletion
│       │
│       ├── package.json
│       ├── tsconfig.json
│       └── README.md
│
├── docs/                            ← This documentation lives here
│   ├── DESIGN.md                    ← This file
│   ├── YAML-REFERENCE.md            ← Complete YAML key reference
│   ├── ARCHITECTURE.md              ← Technical deep-dive
│   ├── CONTRIBUTING.md              ← How to add new modules
│   └── CHANGELOG.md
│
├── examples/                        ← Example YAML configs and their CSS output
│   ├── minimal/
│   ├── full-featured/
│   └── dark-mode/
│
└── package.json                     ← Root (workspace config if monorepo later)
```

---

## 13. YAML Schema Design

The YAML schema is the developer's primary interface. Its design must be learnable, predictable, and self-documenting.

### Design Principles for the Schema

**Principle 1 — Flat where possible, nested where logical.**
`colors.primary` makes sense as nested. `output-format` as a flat key doesn't need to be `output.format.type`.

**Principle 2 — Consistent key naming.** All keys are `kebab-case`. Never `camelCase` or `snake_case` in YAML.

**Principle 3 — Boolean flags for enabling/disabling modules.** `components.button.enabled: false` should completely exclude button CSS from the output.

**Principle 4 — Token references use a clear, unambiguous syntax.** We adopt `{tokens.path.to.value}` with curly braces as the reference syntax. No ambiguity, easy to parse.

**Principle 5 — The schema ships with JSON Schema for IDE support.** VSCode and other editors can validate YAML and provide autocompletion if we ship a `styforge.schema.json`. This is non-negotiable for DX.

### Annotated Schema Skeleton

```yaml
# styforge.yaml

# ─────────────────────────────────────────────
# META — Project identity. All optional.
# ─────────────────────────────────────────────
meta:
  name: "my-project"           # Used in output file header comment
  version: "1.0.0"
  author: "Your Name"

# ─────────────────────────────────────────────
# OUTPUT — Controls what gets written to disk
# ─────────────────────────────────────────────
output:
  dir: "./dist"                # Output directory
  filename: "styles.css"       # Output filename (single mode)
  format: single               # single | split
  minify: false                # Minify output CSS
  prefix: ""                   # Class prefix: "sf-" → .sf-btn

# ─────────────────────────────────────────────
# LAYERS — Controls @layer generation and order
# ─────────────────────────────────────────────
layers:
  enabled:
    - tokens
    - reset
    - base
    - components
    - utilities
  # Order determines CSS cascade precedence (later = higher precedence)
  # Default order matches enabled list

# ─────────────────────────────────────────────
# TOKENS — The design token system
# ─────────────────────────────────────────────
tokens:

  colors:
    # Raw palette — these are not semantic, just named values
    primary:
      50: "#EFF6FF"
      100: "#DBEAFE"
      500: "#3B82F6"
      600: "#2563EB"
      900: "#1E3A8A"
    gray:
      # Shorthand: provide a midpoint and generate a full scale
      generate: true
      base: "#6B7280"
      steps: 10           # generates 50, 100, 200 ... 900, 950
    white: "#FFFFFF"
    black: "#000000"

  spacing:
    base: 4               # Base unit in px. All spacing is a multiple of this.
    scale:
      # Option 1: Named multipliers
      - { name: "1", multiplier: 1 }    # → 4px
      - { name: "2", multiplier: 2 }    # → 8px
      - { name: "4", multiplier: 4 }    # → 16px
      - { name: "6", multiplier: 6 }    # → 24px
      - { name: "8", multiplier: 8 }    # → 32px
      - { name: "12", multiplier: 12 }  # → 48px
      - { name: "16", multiplier: 16 }  # → 64px
      # Option 2: Use a named scale (generates above automatically)
      # preset: "tailwind"   # or "compact" | "spacious" | "fibonacci"

  typography:
    families:
      sans:
        stack: "Inter, system-ui, -apple-system, sans-serif"
        weights: [400, 500, 600, 700]
      serif:
        stack: "Georgia, 'Times New Roman', serif"
        weights: [400, 700]
      mono:
        stack: "'JetBrains Mono', 'Fira Code', monospace"
        weights: [400]
      # Which family is the default body font
      default: sans

    sizes:
      base: 16                  # Base font size in px
      scale: major-third        # Modular scale: major-third | perfect-fourth | golden | custom
      # custom scale:
      # - { name: "xs", value: 12 }
      # - { name: "sm", value: 14 }
      # ...

    line-heights:
      tight: 1.25
      normal: 1.5
      relaxed: 1.75

  radius:
    none: 0
    sm: 2px
    md: 4px
    lg: 8px
    xl: 16px
    full: 9999px

  shadows:
    sm: "0 1px 2px rgba(0,0,0,0.05)"
    md: "0 4px 6px rgba(0,0,0,0.07), 0 1px 3px rgba(0,0,0,0.06)"
    lg: "0 10px 15px rgba(0,0,0,0.1), 0 4px 6px rgba(0,0,0,0.05)"

  transitions:
    fast: 150ms
    normal: 250ms
    slow: 400ms
    easing:
      default: "cubic-bezier(0.4, 0, 0.2, 1)"
      in: "cubic-bezier(0.4, 0, 1, 1)"
      out: "cubic-bezier(0, 0, 0.2, 1)"

  breakpoints:
    sm: 640px
    md: 768px
    lg: 1024px
    xl: 1280px
    2xl: 1536px

# ─────────────────────────────────────────────
# THEMES — Semantic token mappings per theme
# ─────────────────────────────────────────────
themes:
  default: light

  light:
    colors:
      background: "{tokens.colors.white}"
      surface: "{tokens.colors.gray.50}"
      border: "{tokens.colors.gray.200}"
      text:
        primary: "{tokens.colors.gray.900}"
        secondary: "{tokens.colors.gray.600}"
        muted: "{tokens.colors.gray.400}"
      accent:
        primary: "{tokens.colors.primary.600}"
        primary-hover: "{tokens.colors.primary.700}"
        primary-contrast: "{tokens.colors.white}"

  dark:
    colors:
      background: "{tokens.colors.gray.950}"
      surface: "{tokens.colors.gray.900}"
      border: "{tokens.colors.gray.700}"
      text:
        primary: "{tokens.colors.gray.50}"
        secondary: "{tokens.colors.gray.400}"
        muted: "{tokens.colors.gray.600}"
      accent:
        primary: "{tokens.colors.primary.400}"
        primary-hover: "{tokens.colors.primary.300}"
        primary-contrast: "{tokens.colors.gray.950}"

# ─────────────────────────────────────────────
# RESET — Base CSS reset
# ─────────────────────────────────────────────
reset:
  enabled: true
  type: modern               # modern | normalize | opinionated | none
  box-sizing: border-box
  smooth-scroll: true

# ─────────────────────────────────────────────
# BASE — Core HTML element styles
# ─────────────────────────────────────────────
base:
  enabled: true
  body:
    font-family: "{tokens.typography.families.default}"
    font-size: "{tokens.typography.sizes.base}"
    line-height: "{tokens.typography.line-heights.normal}"
    color: "{themes.light.colors.text.primary}"
    background: "{themes.light.colors.background}"
  headings:
    font-family: "{tokens.typography.families.default}"
    font-weight: 700
    line-height: "{tokens.typography.line-heights.tight}"
  links:
    color: "{themes.light.colors.accent.primary}"
    hover-color: "{themes.light.colors.accent.primary-hover}"
    decoration: underline

# ─────────────────────────────────────────────
# COMPONENTS — Optional component CSS generation
# ─────────────────────────────────────────────
components:
  enabled: true               # Master switch for all components

  button:
    enabled: true
    variants: [primary, secondary, ghost, danger, success]
    sizes:
      sm: { padding: "{tokens.spacing.1} {tokens.spacing.2}", font-size: "{tokens.typography.sizes.sm}" }
      md: { padding: "{tokens.spacing.2} {tokens.spacing.4}", font-size: "{tokens.typography.sizes.base}" }
      lg: { padding: "{tokens.spacing.3} {tokens.spacing.6}", font-size: "{tokens.typography.sizes.lg}" }
    radius: "{tokens.radius.md}"
    transition: "{tokens.transitions.fast}"

  card:
    enabled: true
    padding: "{tokens.spacing.6}"
    radius: "{tokens.radius.lg}"
    shadow: "{tokens.shadows.md}"
    border: true

  form:
    enabled: true
    elements: [input, select, textarea, checkbox, radio, label]
    radius: "{tokens.radius.md}"

  badge:
    enabled: true
    variants: [default, primary, success, warning, danger]

  alert:
    enabled: true
    variants: [info, success, warning, danger]

  nav:
    enabled: true

# ─────────────────────────────────────────────
# UTILITIES — Optional utility class generation
# ─────────────────────────────────────────────
utilities:
  enabled: false              # OFF by default. This is not a Tailwind clone.
  include:
    - display                 # .flex, .grid, .block, .hidden
    - spacing                 # .p-{n}, .m-{n}, .gap-{n}
    - text                    # .text-{size}, .font-{weight}
    - colors                  # .text-{color}, .bg-{color}
```

---

## 14. TypeScript Pipeline Design

### Core Types

```typescript
// The compiled, validated, resolved config
interface StyForgeConfig {
  meta: MetaConfig;
  output: OutputConfig;
  layers: LayersConfig;
  tokens: TokenTree;
  themes: ThemeMap;
  reset: ResetConfig;
  base: BaseConfig;
  components: ComponentsConfig;
  utilities: UtilitiesConfig;
}

// Resolved token tree — all references resolved, all scales expanded
interface TokenTree {
  colors: Record<string, Record<string, string> | string>;
  spacing: Record<string, string>;
  typography: TypographyTokens;
  radius: Record<string, string>;
  shadows: Record<string, string>;
  transitions: TransitionTokens;
  breakpoints: Record<string, string>;
}

// The shared context passed to every module
interface BuildContext {
  config: StyForgeConfig;
  tokens: ResolvedTokenTree;     // Fully resolved (no {token.refs} remaining)
  themes: ResolvedThemeMap;
  logger: Logger;
}

// Every module implements this interface
interface StyForgeModule {
  name: string;
  phase: 'tokens' | 'reset' | 'base' | 'components' | 'utilities';
  generate(context: BuildContext): string;  // Returns CSS string
}
```

### Config Validation (Zod)

```typescript
// All config validation is done with Zod
// This gives us:
// 1. Runtime type safety
// 2. Detailed, human-readable error messages
// 3. Auto-generated TypeScript types
// 4. The ability to generate a JSON Schema for IDE support

import { z } from 'zod';

const ColorValueSchema = z.string().regex(/^#[0-9A-Fa-f]{3,8}$/, 
  "Color values must be valid hex codes"
);

const SpacingScaleSchema = z.object({
  base: z.number().int().positive().default(4),
  scale: z.array(z.object({
    name: z.string(),
    multiplier: z.number().positive()
  })).optional()
});

// ... Full schema: every key validated, every error useful
```

### Module Registration Pattern

```typescript
// Modules self-register. The pipeline discovers them.
// Adding a new component = create a new module file. Zero changes to core.

class BuildPipeline {
  private modules: Map<string, StyForgeModule> = new Map();

  register(module: StyForgeModule): this {
    this.modules.set(module.name, module);
    return this;
  }

  async run(config: StyForgeConfig): Promise<string> {
    const context = await this.buildContext(config);
    const cssBlocks: string[] = [];

    for (const layerName of config.layers.enabled) {
      const layerModules = [...this.modules.values()]
        .filter(m => m.phase === layerName);

      for (const module of layerModules) {
        const css = module.generate(context);
        if (css.trim()) cssBlocks.push(css);
      }
    }

    return this.assembler.assemble(cssBlocks, config);
  }
}
```

---

## 15. Developer Experience (DX) Goals

DX is a first-class concern. These are measurable goals, not aspirations.

**Goal 1 — Time to first CSS: under 2 minutes.**
A developer should be able to run `npx styforge init`, open the generated `styforge.yaml`, run `npx styforge build`, and have a working CSS file in under 2 minutes. No reading docs required for the basic case.

**Goal 2 — Errors are actionable.**
When a validation error occurs, the error message tells the developer exactly what is wrong, where it is in the YAML file, and how to fix it. No cryptic stack traces. Example:
```
✖ Config error in styforge.yaml line 14:
  tokens.colors.primary.500: "#3G82F6" is not a valid hex color.
  Expected format: #RRGGBB or #RGB
  Got: "#3G82F6"
```

**Goal 3 — IDE first-class support.**
The shipped JSON Schema means VSCode and JetBrains users get autocompletion and inline validation in their YAML file. This should be documented in the README with a one-line setup instruction.

**Goal 4 — The output file is human-readable.**
The generated CSS is clean, well-commented, and organized. A developer should be able to open `styles.css` and immediately understand its structure, even if they never read a doc.

**Goal 5 — Watch mode is fast.**
`styforge watch` should rebuild in under 100ms on any reasonable config. CSS compilation is not computationally expensive, and we should never make it feel slow.

**Goal 6 — Presets lower the barrier to entry.**
`styforge init --preset minimal` gives a developer a complete, working, beautiful config in one command. Presets are curated starting points, not lock-in.

---

## 16. Open Questions

These are the questions that cannot be answered in documentation alone — they require decisions from the project owner or will be resolved through implementation experience.

| # | Question | Impact | Decision Needed By |
|---|----------|--------|-------------------|
| 1 | Should component CSS include `:hover`, `:focus`, `:disabled` states by default, or should those be opt-in? | DX, output size | Before component module implementation |
| 2 | Should the tool support custom user-defined modules? (i.e., a plugin system for end-users, not just internal modules) | Architecture scope | Before pipeline finalization |
| 3 | Should `styforge watch` use file system watching (chokidar) or a more lightweight polling approach? | DX, dependencies | Before CLI implementation |
| 4 | What is the versioning strategy for the YAML schema itself? If a key is renamed between v1.0 and v2.0, how is migration handled? | Long-term maintainability | Before v1.0 release |
| 5 | Should we support `@import` in output CSS for split mode, or leave includes to the developer? | Output format | Before output module implementation |
| 6 | Should color palette generation (from a base color) use HSL math, OKLCH, or a simpler algorithm? | Color quality | Before token compiler implementation |
| 7 | What is the minimum Node.js version target? (This affects which TypeScript/ESM features are available) | Build constraints | Before package.json is written |
| 8 | Should presets be built-in to the package or fetched from a registry (like a "preset marketplace")? | Distribution scope | Before first preset is authored |

---

*Document maintained by the project owner. All architectural decisions are recorded here before implementation begins. This document is the source of truth for "why was this built this way."*
