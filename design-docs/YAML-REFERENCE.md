# StyForge — YAML Reference

> Complete reference for every key in `styforge.yaml`.  
> All keys are optional unless marked **required**.  
> Keys use `kebab-case` convention throughout.

---

## Quick Start

The simplest valid config (everything else uses defaults):

```yaml
tokens:
  colors:
    primary: "#3B82F6"
```

A production-ready config:

```yaml
meta:
  name: "my-project"

tokens:
  colors:
    primary:
      500: "#3B82F6"
      600: "#2563EB"
  typography:
    families:
      sans:
        stack: "Inter, system-ui, sans-serif"

themes:
  light:
    colors:
      background: "#FFFFFF"
      text-primary: "{tokens.colors.gray.900}"
  dark:
    colors:
      background: "#0F172A"
      text-primary: "{tokens.colors.gray.50}"

components:
  button:
    enabled: true
    variants: [primary, secondary, ghost]
```

---

## Token References

Values anywhere in the YAML can reference other values using the `{path.to.value}` syntax.

```yaml
themes:
  light:
    colors:
      background: "{tokens.colors.white}"        # ✓ References raw token
      accent: "{tokens.colors.primary.500}"      # ✓ References palette shade
      surface: "#F9FAFB"                          # ✓ Direct value also valid
```

Rules:
- References must point to an existing value in the final merged config
- Circular references are detected and reported as errors
- References are resolved before any CSS is generated
- If a reference cannot be resolved, the build fails with a clear error

---

## `meta` — Project Identity

```yaml
meta:
  name: "my-project"       # Appears in output file header comment
  version: "1.0.0"         # Appears in output file header comment
  author: "Your Name"      # Appears in output file header comment
```

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `name` | string | `"styforge-project"` | Project name |
| `version` | string | `"1.0.0"` | Project version |
| `author` | string | `""` | Author name |

---

## `output` — Build Output

```yaml
output:
  dir: "./dist"
  filename: "styles.css"
  format: single
  minify: false
  prefix: ""
  source-map: false
```

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `dir` | string | `"./dist"` | Output directory (relative to config file) |
| `filename` | string | `"styles.css"` | Output filename (used when `format: single`) |
| `format` | `single` \| `split` | `single` | `single`: one CSS file. `split`: separate files per layer |
| `minify` | boolean | `false` | Minify the output CSS |
| `prefix` | string | `""` | Class name prefix. `"sf-"` → `.sf-btn`, `.sf-card` |
| `source-map` | boolean | `false` | Generate a source map file |

### Split Format Output

When `format: split`, files are named by layer:
```
dist/
  tokens.css
  reset.css
  base.css
  components.css
  utilities.css
  index.css     ← @imports all of the above in order
```

---

## `layers` — CSS `@layer` Control

```yaml
layers:
  enabled:
    - tokens
    - reset
    - base
    - components
    - utilities
```

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | string[] | `[tokens, reset, base, components]` | Which layers to generate. Order determines CSS cascade precedence (later = wins). |

Available layer names: `tokens`, `reset`, `base`, `components`, `utilities`

**Why this matters:** CSS `@layer` means that any CSS you write *outside* these layers (in your own stylesheets) will automatically take precedence over all layer content, regardless of specificity. This is the escape hatch — you never fight generated CSS.

---

## `tokens` — Design Tokens

### `tokens.colors`

#### Palette colors (named shades)
```yaml
tokens:
  colors:
    primary:
      50: "#EFF6FF"
      100: "#DBEAFE"
      200: "#BFDBFE"
      300: "#93C5FD"
      400: "#60A5FA"
      500: "#3B82F6"
      600: "#2563EB"
      700: "#1D4ED8"
      800: "#1E40AF"
      900: "#1E3A8A"
      950: "#172554"
```

Generated CSS custom properties:
```css
:root {
  --color-primary-50: #EFF6FF;
  --color-primary-100: #DBEAFE;
  /* ... etc */
  --color-primary-950: #172554;
}
```

#### Auto-generated palette
```yaml
tokens:
  colors:
    brand:
      generate: true     # Auto-generate full palette
      base: "#7C3AED"    # The base color (will become the 500 shade)
      steps: 10          # Number of shades (generates: 50, 100-900, 950)
      algorithm: oklch   # oklch (default, perceptually uniform) | hsl
```

#### Flat colors (no shades needed)
```yaml
tokens:
  colors:
    white: "#FFFFFF"
    black: "#000000"
    transparent: "transparent"
```

Generated: `--color-white: #FFFFFF;`

---

### `tokens.spacing`

#### Using a preset scale
```yaml
tokens:
  spacing:
    base: 4           # Base unit in px
    preset: tailwind  # Generates the full Tailwind-compatible spacing scale
```

Available presets:
| Preset | Description | Sample values |
|--------|-------------|---------------|
| `tailwind` | Standard 4px-based scale | 1→4px, 2→8px, 4→16px, 8→32px, 16→64px |
| `compact` | Smaller steps for dense UIs | 1→2px, 2→4px, 3→6px, 4→8px |
| `spacious` | Larger steps for open layouts | 1→8px, 2→16px, 3→24px, 4→32px |
| `fibonacci` | Fibonacci × base unit | 4, 8, 12, 20, 32, 52, 84, 136px |

#### Custom scale
```yaml
tokens:
  spacing:
    base: 4
    scale:
      - name: "px"
        value: 1px      # Can also use direct value instead of multiplier
      - name: "0.5"
        multiplier: 0.5  # → 2px
      - name: "1"
        multiplier: 1    # → 4px
      - name: "2"
        multiplier: 2    # → 8px
      - name: "4"
        multiplier: 4    # → 16px
```

Generated CSS:
```css
:root {
  --spacing-px: 1px;
  --spacing-0-5: 0.5rem;   /* Values converted to rem */
  --spacing-1: 0.25rem;
  --spacing-2: 0.5rem;
  --spacing-4: 1rem;
}
```

Note: Pixel values are converted to `rem` using `base ÷ 16`. Exception: values ≤ 2px remain as `px`.

---

### `tokens.typography`

#### Font families
```yaml
tokens:
  typography:
    families:
      sans:
        stack: "Inter, system-ui, -apple-system, sans-serif"
        weights: [400, 500, 600, 700]
      serif:
        stack: "Georgia, 'Times New Roman', serif"
        weights: [400, 700]
      mono:
        stack: "'JetBrains Mono', 'Fira Code', Menlo, monospace"
        weights: [400]
      display:
        stack: "'Playfair Display', Georgia, serif"
        weights: [700, 900]
      default: sans    # Which family is applied to body/html
```

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `stack` | string | — | CSS font-family value |
| `weights` | number[] | `[400, 700]` | Loaded font weights (informational — does not generate @font-face) |
| `default` | string | `sans` | Key of the default body font family |

#### Font sizes

Using a modular scale:
```yaml
tokens:
  typography:
    sizes:
      base: 16          # Base size in px
      scale: major-third  # Modular scale ratio
```

Available scales:
| Scale name | Ratio | Use case |
|-----------|-------|---------|
| `minor-second` | 1.067 | Very subtle hierarchy |
| `major-second` | 1.125 | Subtle, works at small sizes |
| `minor-third` | 1.200 | Balanced for tight UIs |
| `major-third` | 1.250 | **Recommended** — clear but not dramatic |
| `perfect-fourth` | 1.333 | Editorial, good for blog/docs |
| `augmented-fourth` | 1.414 | Bold hierarchy |
| `golden` | 1.618 | Very dramatic — landing pages |

Custom sizes:
```yaml
tokens:
  typography:
    sizes:
      custom:
        - name: "xs"
          value: 12
        - name: "sm"
          value: 14
        - name: "base"
          value: 16
        - name: "lg"
          value: 18
        - name: "xl"
          value: 20
        - name: "2xl"
          value: 24
        - name: "3xl"
          value: 30
        - name: "4xl"
          value: 36
```

#### Line heights
```yaml
tokens:
  typography:
    line-heights:
      none: 1
      tight: 1.25
      snug: 1.375
      normal: 1.5      # Default for body text
      relaxed: 1.625
      loose: 2
```

#### Letter spacing
```yaml
tokens:
  typography:
    letter-spacing:
      tighter: "-0.05em"
      tight: "-0.025em"
      normal: "0em"
      wide: "0.025em"
      wider: "0.05em"
      widest: "0.1em"
```

---

### `tokens.radius`

```yaml
tokens:
  radius:
    none: 0
    sm: "2px"
    md: "4px"
    lg: "8px"
    xl: "12px"
    2xl: "16px"
    3xl: "24px"
    full: "9999px"
```

---

### `tokens.shadows`

```yaml
tokens:
  shadows:
    none: "none"
    sm: "0 1px 2px 0 rgba(0, 0, 0, 0.05)"
    md: "0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -2px rgba(0, 0, 0, 0.1)"
    lg: "0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -4px rgba(0, 0, 0, 0.1)"
    xl: "0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 8px 10px -6px rgba(0, 0, 0, 0.1)"
    2xl: "0 25px 50px -12px rgba(0, 0, 0, 0.25)"
    inner: "inset 0 2px 4px 0 rgba(0, 0, 0, 0.05)"
```

---

### `tokens.transitions`

```yaml
tokens:
  transitions:
    duration:
      instant: "0ms"
      fast: "150ms"
      normal: "250ms"
      slow: "400ms"
      slower: "700ms"
    easing:
      linear: "linear"
      default: "cubic-bezier(0.4, 0, 0.2, 1)"   # ease-in-out
      in: "cubic-bezier(0.4, 0, 1, 1)"           # ease-in
      out: "cubic-bezier(0, 0, 0.2, 1)"          # ease-out
      bounce: "cubic-bezier(0.34, 1.56, 0.64, 1)"
```

---

### `tokens.breakpoints`

```yaml
tokens:
  breakpoints:
    sm: "640px"
    md: "768px"
    lg: "1024px"
    xl: "1280px"
    2xl: "1536px"
```

Generated CSS:
```css
:root {
  --breakpoint-sm: 640px;
  --breakpoint-md: 768px;
  /* etc */
}
```

Note: Breakpoints as CSS custom properties cannot be used directly in `@media` queries (CSS limitation). They are available as tokens for documentation/reference purposes. The build pipeline uses them internally for responsive component variants.

---

## `themes` — Semantic Token Mapping

Themes map semantic role names to raw token values. Semantic tokens are what you use in your HTML — they describe *what a value is for*, not what its color is.

```yaml
themes:
  default: light     # Which theme is active on :root by default

  light:
    colors:
      # Surface colors
      background: "{tokens.colors.white}"
      surface: "{tokens.colors.gray.50}"
      surface-raised: "{tokens.colors.white}"
      overlay: "{tokens.colors.gray.100}"

      # Border colors
      border: "{tokens.colors.gray.200}"
      border-strong: "{tokens.colors.gray.400}"

      # Text colors
      text-primary: "{tokens.colors.gray.900}"
      text-secondary: "{tokens.colors.gray.600}"
      text-muted: "{tokens.colors.gray.400}"
      text-disabled: "{tokens.colors.gray.300}"
      text-inverse: "{tokens.colors.white}"

      # Accent / interactive colors
      accent: "{tokens.colors.primary.600}"
      accent-hover: "{tokens.colors.primary.700}"
      accent-active: "{tokens.colors.primary.800}"
      accent-subtle: "{tokens.colors.primary.50}"
      accent-contrast: "{tokens.colors.white}"

      # Feedback colors
      success: "#16A34A"
      success-subtle: "#F0FDF4"
      warning: "#D97706"
      warning-subtle: "#FFFBEB"
      danger: "#DC2626"
      danger-subtle: "#FEF2F2"
      info: "#2563EB"
      info-subtle: "#EFF6FF"

  dark:
    colors:
      background: "{tokens.colors.gray.950}"
      surface: "{tokens.colors.gray.900}"
      surface-raised: "{tokens.colors.gray.800}"
      overlay: "{tokens.colors.gray.800}"
      border: "{tokens.colors.gray.700}"
      border-strong: "{tokens.colors.gray.500}"
      text-primary: "{tokens.colors.gray.50}"
      text-secondary: "{tokens.colors.gray.400}"
      text-muted: "{tokens.colors.gray.600}"
      text-disabled: "{tokens.colors.gray.700}"
      text-inverse: "{tokens.colors.gray.900}"
      accent: "{tokens.colors.primary.400}"
      accent-hover: "{tokens.colors.primary.300}"
      accent-active: "{tokens.colors.primary.200}"
      accent-subtle: "#1e3a5f"
      accent-contrast: "{tokens.colors.gray.950}"
      # Feedback colors can also reference tokens or be direct values
      success: "#4ADE80"
      success-subtle: "#052E16"
      warning: "#FCD34D"
      warning-subtle: "#1C1400"
      danger: "#F87171"
      danger-subtle: "#1C0A0A"
      info: "#60A5FA"
      info-subtle: "#0C1E3D"
```

Generated CSS:
```css
:root {
  --color-background: #FFFFFF;
  --color-text-primary: #111827;
  --color-accent: #2563EB;
  /* ... etc */
}

[data-theme="dark"] {
  --color-background: #030712;
  --color-text-primary: #F9FAFB;
  --color-accent: #60A5FA;
  /* ... etc */
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --color-background: #030712;
    /* ... */
  }
}
```

---

## `reset` — CSS Reset

```yaml
reset:
  enabled: true
  type: modern
  box-sizing: border-box
  smooth-scroll: true
```

| Key | Type | Default | Options | Description |
|-----|------|---------|---------|-------------|
| `enabled` | boolean | `true` | — | Include a reset in the output |
| `type` | string | `modern` | `modern`, `normalize`, `opinionated`, `none` | Which reset strategy to use |
| `box-sizing` | string | `border-box` | `border-box`, `content-box` | Applied to `*` |
| `smooth-scroll` | boolean | `true` | — | Adds `scroll-behavior: smooth` to `:root` (respects `prefers-reduced-motion`) |

### Reset Types

**`modern`** — A minimal, modern reset (~30 lines). Removes default margins/padding, sets `box-sizing`, resets form elements, improves image/media handling. No opinions beyond fundamentals.

**`normalize`** — Based on Normalize.css. Makes browsers render elements more consistently without removing all defaults. Good for projects where default browser styles are acceptable.

**`opinionated`** — The modern reset plus: removes list styles on `<ul>`/`<ol>`, better typography defaults, improved interactive element cursor styles, better scroll behavior. Good for apps.

**`none`** — No reset. Use this if you're providing your own reset or working in a context where a reset would conflict.

---

## `base` — HTML Element Styles

```yaml
base:
  enabled: true

  body:
    font-family: "{tokens.typography.families.default}"
    font-size: "{tokens.typography.sizes.base}"
    line-height: "{tokens.typography.line-heights.normal}"
    color: "{themes.light.colors.text-primary}"
    background: "{themes.light.colors.background}"
    -webkit-font-smoothing: antialiased

  headings:
    font-family: "{tokens.typography.families.default}"
    font-weight: 700
    line-height: "{tokens.typography.line-heights.tight}"
    color: "{themes.light.colors.text-primary}"
    margin-top: "{tokens.spacing.6}"
    margin-bottom: "{tokens.spacing.3}"
    # Individual heading overrides:
    h1:
      font-size: "{tokens.typography.sizes.4xl}"
    h2:
      font-size: "{tokens.typography.sizes.3xl}"
    h3:
      font-size: "{tokens.typography.sizes.2xl}"
    h4:
      font-size: "{tokens.typography.sizes.xl}"
    h5:
      font-size: "{tokens.typography.sizes.lg}"
    h6:
      font-size: "{tokens.typography.sizes.base}"

  links:
    color: "{themes.light.colors.accent}"
    hover-color: "{themes.light.colors.accent-hover}"
    decoration: underline
    hover-decoration: underline

  code:
    font-family: "{tokens.typography.families.mono}"
    font-size: "0.875em"
    background: "{themes.light.colors.surface}"
    padding: "0.125em 0.25em"
    radius: "{tokens.radius.sm}"

  hr:
    border-color: "{themes.light.colors.border}"
    margin: "{tokens.spacing.8} 0"
```

---

## `components` — Component CSS

### Master Switch

```yaml
components:
  enabled: true   # false disables ALL components regardless of individual settings
```

### `components.button`

```yaml
components:
  button:
    enabled: true
    variants:
      - primary
      - secondary
      - ghost
      - danger
      - success
      - warning
      - link          # Looks like a text link
    sizes:
      sm:
        padding: "{tokens.spacing.1} {tokens.spacing.3}"
        font-size: "{tokens.typography.sizes.sm}"
        height: "32px"
      md:
        padding: "{tokens.spacing.2} {tokens.spacing.4}"
        font-size: "{tokens.typography.sizes.base}"
        height: "40px"
      lg:
        padding: "{tokens.spacing.3} {tokens.spacing.6}"
        font-size: "{tokens.typography.sizes.lg}"
        height: "48px"
    default-size: md
    radius: "{tokens.radius.md}"
    font-weight: 500
    transition: "{tokens.transitions.duration.fast}"
    # Generate icon-only (square) variants
    icon-button: true
```

Generated classes: `.btn`, `.btn--primary`, `.btn--secondary`, `.btn--sm`, `.btn--lg`, `.btn--icon`

### `components.card`

```yaml
components:
  card:
    enabled: true
    padding: "{tokens.spacing.6}"
    radius: "{tokens.radius.lg}"
    shadow: "{tokens.shadows.md}"
    border: true
    border-color: "{themes.light.colors.border}"
    background: "{themes.light.colors.surface-raised}"
    # Sub-elements
    header:
      padding-bottom: "{tokens.spacing.4}"
      border-bottom: true
    footer:
      padding-top: "{tokens.spacing.4}"
      border-top: true
```

Generated classes: `.card`, `.card__header`, `.card__body`, `.card__footer`

### `components.form`

```yaml
components:
  form:
    enabled: true
    elements:
      - input
      - textarea
      - select
      - checkbox
      - radio
      - label
      - fieldset
    input:
      height: "40px"
      padding: "{tokens.spacing.2} {tokens.spacing.3}"
      radius: "{tokens.radius.md}"
      border-color: "{themes.light.colors.border}"
      focus-ring-color: "{themes.light.colors.accent}"
      background: "{themes.light.colors.background}"
      font-size: "{tokens.typography.sizes.base}"
```

Generated classes: `.input`, `.textarea`, `.select`, `.label`, `.checkbox`, `.radio`, `.fieldset`, `.form-group`, `.form-hint`, `.form-error`

### `components.badge`

```yaml
components:
  badge:
    enabled: true
    variants: [default, primary, success, warning, danger, info]
    sizes: [sm, md]
    radius: "{tokens.radius.full}"
```

Generated classes: `.badge`, `.badge--primary`, `.badge--success`, `.badge--sm`

### `components.alert`

```yaml
components:
  alert:
    enabled: true
    variants: [info, success, warning, danger]
    radius: "{tokens.radius.md}"
    # Include icon slot styling
    has-icon: true
```

Generated classes: `.alert`, `.alert--info`, `.alert--success`, `.alert__icon`, `.alert__message`, `.alert__title`

### `components.nav`

```yaml
components:
  nav:
    enabled: true
    # Horizontal nav bar
    horizontal:
      enabled: true
      height: "64px"
    # Vertical sidebar nav
    vertical:
      enabled: true
```

Generated classes: `.nav`, `.nav__brand`, `.nav__links`, `.nav__item`, `.nav__item--active`

---

## `utilities` — Utility Classes (Optional)

**Default: disabled.** This is not a Tailwind replacement. Enable only if you need a handful of layout helpers in a project that doesn't use a utility framework.

```yaml
utilities:
  enabled: false
  include:
    - display          # .flex, .grid, .block, .inline, .hidden
    - flex             # .flex-col, .items-center, .justify-between, .gap-{n}
    - spacing          # .p-{n}, .px-{n}, .py-{n}, .m-{n}, .mx-auto
    - text             # .text-{size}, .font-{weight}, .text-center, .truncate
    - width            # .w-full, .w-auto, .max-w-{n}
    - overflow         # .overflow-hidden, .overflow-auto
    - position         # .relative, .absolute, .fixed, .sticky
    - z-index          # .z-{0,10,20,30,40,50}
    - sr-only          # .sr-only (screen reader only)
```

---

## Presets

Presets are pre-built YAML configs you can use as a starting point:

```bash
styforge init --preset minimal
styforge init --preset editorial
styforge init --preset dashboard
styforge init --preset dark-first
```

| Preset | Description |
|--------|-------------|
| `minimal` | Clean, minimal tokens. Barely opinionated. A blank canvas. |
| `editorial` | Large type scale, serif options, generous spacing. For blogs and docs. |
| `dashboard` | Compact spacing, mono font available, dense component variants. For apps. |
| `dark-first` | Dark as the default theme. Muted palette. For creative/dev tools. |

To see available presets: `styforge list presets`

---

## Complete Example: `styforge.yaml`

```yaml
meta:
  name: "acme-website"
  version: "2.0.0"

output:
  dir: "./public/css"
  filename: "styles.css"
  minify: true

tokens:
  colors:
    primary:
      generate: true
      base: "#7C3AED"
    gray:
      generate: true
      base: "#6B7280"
    white: "#FFFFFF"
    black: "#000000"

  spacing:
    base: 4
    preset: tailwind

  typography:
    families:
      sans:
        stack: "'DM Sans', system-ui, sans-serif"
        weights: [400, 500, 700]
      mono:
        stack: "'JetBrains Mono', monospace"
        weights: [400]
      default: sans
    sizes:
      base: 16
      scale: major-third
    line-heights:
      tight: 1.25
      normal: 1.5

  radius:
    sm: 4px
    md: 8px
    lg: 12px
    full: 9999px

  shadows:
    sm: "0 1px 3px rgba(0,0,0,0.1)"
    md: "0 4px 12px rgba(0,0,0,0.1)"

  transitions:
    duration:
      fast: 150ms
      normal: 250ms
    easing:
      default: "cubic-bezier(0.4, 0, 0.2, 1)"

themes:
  default: light
  light:
    colors:
      background: "{tokens.colors.white}"
      surface: "#F5F3FF"
      border: "#E5E7EB"
      text-primary: "#111827"
      text-secondary: "#6B7280"
      accent: "{tokens.colors.primary.600}"
      accent-hover: "{tokens.colors.primary.700}"
      accent-contrast: "{tokens.colors.white}"
  dark:
    colors:
      background: "#0F0A1A"
      surface: "#1A1030"
      border: "{tokens.colors.primary.900}"
      text-primary: "#F9FAFB"
      text-secondary: "{tokens.colors.gray.400}"
      accent: "{tokens.colors.primary.400}"
      accent-hover: "{tokens.colors.primary.300}"
      accent-contrast: "#0F0A1A"

reset:
  type: opinionated

components:
  button:
    enabled: true
    variants: [primary, secondary, ghost]
    sizes:
      sm: { padding: "6px 12px", font-size: "0.875rem" }
      md: { padding: "10px 20px", font-size: "1rem" }
      lg: { padding: "14px 28px", font-size: "1.125rem" }
  card:
    enabled: true
  form:
    enabled: true
    elements: [input, select, textarea, label]
  badge:
    enabled: true
    variants: [default, primary, success, danger]
  alert:
    enabled: false   # Not needed for this project
```
