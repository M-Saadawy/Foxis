# StyForge — Vision & Philosophy

> *"The best tool is the one you don't have to think about."*

---

## The Vision in One Sentence

StyForge is a YAML-to-CSS compiler that lets developers describe their design system once and carry it to every project — with zero runtime cost and zero framework lock-in.

---

## The Problem With the Current Landscape

### The Two Extremes

The CSS tooling ecosystem has evolved into two opposite extremes, with almost nothing in between:

**Extreme 1: Full frameworks (Bootstrap, Foundation)**
These ship a complete, opinionated visual system. They're fast to start, but you spend the next six months overriding their decisions. Your project doesn't look like itself — it looks like the framework.

**Extreme 2: Nothing (starting from scratch)**
You write your own reset. You define your own spacing scale. You create your own button CSS for the fifteenth time. You copy-paste the color palette you used last project. This is the option serious developers often choose — and it has a hidden tax. It costs time. Every. Single. Project.

**Tailwind sits in the middle, but it has an opinion you can't escape:**
Tailwind is genuinely excellent, but it requires a build step in every consuming project, it couples your HTML to your styling framework, and removing it from a project is a significant undertaking. It is also so widely adopted that "Tailwind project" has become a recognizable aesthetic.

### The Hidden Tax

The hidden tax we're eliminating is the *recurring cost of foundational decisions*.

Every developer has a set of CSS decisions they make on every project:
- What's my base spacing unit?
- What's my color palette?
- What's my typography scale?
- What reset do I use?
- What does my button look like?
- How do I handle dark mode?

These aren't creative decisions — they're *maintenance decisions*. A developer who has made them once should never have to make them again from scratch.

StyForge turns these decisions into a config file you own, version control, refine over time, and deploy in seconds to every new project.

---

## What "Enterprise-Grade" Actually Means

"Enterprise-grade" is a phrase that gets misused. In the context of StyForge, it means five specific things:

**1. Reproducibility.** Given the same `styforge.yaml`, the build produces the exact same CSS on any machine, at any point in time. No environmental variability. No random behavior.

**2. Traceability.** The output CSS is structured and commented so that any CSS rule can be traced back to the YAML key that produced it. Debugging means reading the config, not reverse-engineering the output.

**3. Zero surprise upgrades.** When StyForge releases a new version, your config continues to produce the same CSS unless you explicitly opt into breaking changes. The YAML schema is versioned. Migration guides are provided for every breaking change.

**4. Testable output.** The CSS output can be validated and snapshot-tested in a CI pipeline. Regressions are caught before they ship.

**5. Team-ready.** The tool works as well for a team of ten as it does for a solo developer. Config files are human-readable and reviewable in pull requests. The schema is self-documenting. A new developer on the team can understand the entire design system by reading `styforge.yaml`.

---

## Why TypeScript for the Build Tool

The build pipeline is TypeScript not because TypeScript is trendy, but because:

- **Safety.** The pipeline manipulates strings of CSS and complex token trees. Type safety prevents entire categories of bugs — particularly around token references and scale generation.
- **Refactorability.** When the schema changes (and it will), TypeScript catches every place in the codebase that needs to be updated. Refactoring without types in a project this size invites silent regressions.
- **Documentation by types.** The TypeScript interfaces for `StyForgeConfig`, `BuildContext`, `StyForgeModule` are themselves documentation. They're more precise than any prose description of the same concepts.
- **Contributor confidence.** A contributor adding a new component module gets autocompletion and type errors. They can't accidentally return the wrong type or access a non-existent token.

The irony that a tool producing zero-JavaScript output is itself written in TypeScript is acknowledged and intentional. The complexity lives in the build tool, not the output.

---

## Why Plain CSS Output

This is the most important design decision.

Every major CSS tool of the last decade has added a runtime cost:
- CSS-in-JS libraries generate styles at component mount time
- CSS Modules require a build step and module resolution
- Tailwind requires a purge/tree-shaking step in CI
- Even PostCSS requires PostCSS to be in the consuming project

StyForge's output is *already done*. The CSS file is the artifact. It requires:
- No JavaScript engine
- No Node.js
- No npm
- No module bundler
- No build step in the consuming project

A developer can take the output `styles.css` and drop it into:
- A plain HTML file
- A WordPress theme
- A Ruby on Rails app
- A Next.js project
- A static site generator
- An email template (within reason)
- A legacy codebase from 2010

The CSS works everywhere CSS works. That is the point.

---

## The CSS Custom Properties Decision

The token system uses CSS custom properties (CSS variables) as the delivery mechanism. This was not always the obvious choice — in 2018, a tool like this might have generated Sass variables or compiled-in static values.

CSS custom properties are the right choice today because:

**They're live.** Custom properties cascade, inherit, and respond to the DOM. Changing a theme is as simple as swapping a `data-theme` attribute on the `<html>` element. No JavaScript, no class changes, no page reload.

**They're inspectable.** In DevTools, you can see every custom property applied to an element. Debugging a color or spacing issue means reading the computed properties pane — it's all there.

**They're overrideable.** Any CSS written after the generated file can override individual custom properties without specificity battles. The `@layer` system amplifies this — anything outside a named layer wins automatically.

**They're the web platform.** Custom properties are not a build abstraction. They exist in the browser natively. They will not be deprecated. Learning StyForge is also learning a transferable CSS skill.

---

## The `@layer` Decision

CSS Cascade Layers (`@layer`) were introduced in 2022 and are now supported in all modern browsers. They solve the oldest problem in CSS: specificity conflicts.

Before `@layer`, using any CSS framework meant fighting its specificity. Adding `!important` to override a framework's `.btn` style. Writing more specific selectors. Creating specificity wars.

With `@layer`, StyForge's generated CSS lives in named layers:

```css
@layer tokens, reset, base, components, utilities;
```

Any CSS you write *outside* these layers automatically wins, regardless of specificity. You can override `.btn { background: red }` without knowing what specificity StyForge used. The cascade is explicit, not a game.

This is the correct modern approach. StyForge adopts it from day one because retrofitting `@layer` into an existing codebase is more expensive than building with it from the start.

---

## The Naming of Things

The tool is called **StyForge** as a working title. The name reflects:
- *Sty* (Style) — what it produces
- *Forge* — the act of making something durable from raw material

The command is `styforge`. The config file is `styforge.yaml`. The output is CSS.

Naming is a real decision that should happen before public release. A good name is short, memorable, pronounceable, and available as an npm package name.

---

## Roadmap Philosophy

StyForge will resist feature creep from day one.

The core promise — YAML in, CSS out — must remain true at every version. Features that violate this promise (runtime behavior, JavaScript output, framework-specific integrations) are explicitly out of scope.

Features that deepen the core promise are always in scope:
- More built-in components
- Better color generation algorithms
- More modular scale options
- Better error messages
- More presets
- IDE plugins (for YAML validation/autocompletion)

Features that expand the scope with care:
- A `styforge eject` command that produces a fully expanded YAML with all defaults written out (useful for understanding and forking)
- A `styforge diff` command that shows what CSS would change if you modified the config
- Integration guides for specific frameworks (not integrations — just guides)

Features that are explicitly out of scope forever:
- Any runtime JavaScript
- Any CSS framework integration
- Any component markup/HTML generation
- Any design tool integration (Figma, Sketch) — these are separate tools
- Any cloud service or account requirement

---

## The Target Developer

StyForge is built for the developer who:

- Has been building for long enough to have CSS opinions
- Starts new projects regularly enough to feel the recurring setup cost
- Values code quality and maintainability over speed-to-first-pixel
- Wants to own their design system, not rent someone else's
- Works on projects that range from small sites to large applications

It is *not* built for:
- Developers who are happy with Tailwind (great — use Tailwind)
- Developers who are happy starting from scratch every time (valid choice)
- Developers who want a visual drag-and-drop design system tool
- Developers who need pre-designed UI components (not just CSS patterns)

---

*This document is a living statement of intent. It should be read before making any significant architectural or feature decision.*
