---
name: design-system
description: Generate a spec-compliant DESIGN.md (design tokens in YAML front matter + design rationale in markdown) from the existing codebase, images (screenshots, mockups, logos), video, documents (brand guidelines, PDFs), URLs, or a text description — then reference it from README.md, AGENTS.md, CLAUDE.md, and other agent entrypoints. Use when the user wants to create or update a DESIGN.md, extract or document a design system / visual identity, define design tokens, or make AI-generated UI consistent across a project.
version: 1.0.0
license: MIT
---

# DESIGN.md generator and wiring

Produce a spec-compliant `DESIGN.md` from whatever the user provides — the
existing codebase, images, video, documents, URLs, a written vibe — then wire
it into the project's entrypoints so every agent (and human) reads it before
generating UI.

`DESIGN.md` is the design counterpart to `AGENTS.md`:

| File | Who reads it | What it defines |
| --- | --- | --- |
| `README.md` | Humans | What the project is |
| `AGENTS.md` | Coding agents | How to build the project |
| `DESIGN.md` | Design agents | How the project should look and feel |

The format follows the DESIGN.md specification. It has two normative layers:

- **YAML front matter** — machine-readable tokens that agents enforce literally
  (exact hex values, font properties, spacing scales).
- **Markdown body** — human-readable rationale organized into `##` sections.

Tokens are the normative values; prose explains how to apply them. The spec is a
foundation, not a prescription: unknown sections and custom tokens are accepted,
never rejected.

## Workflow

### 1. Gather inputs

Collect three kinds of input, in this order:

- **Codebase context (always checked first).** Before analyzing anything the
  user attached, scan the project for an existing design identity: design
  tokens (`tailwind.config`, CSS custom properties, theme files,
  styled-components theme), fonts already in use, the dominant styling of
  existing components, and any existing `DESIGN.md`. If a coherent identity
  exists, it anchors the output — generated tokens extend it, they do not
  replace it.
- **User-supplied media** — images, video, documents, or URLs (process each
  per *Handling each medium* below). Classify every item:
  - **Project artifact** — a screen, mockup, recording, document, or page *of
    this product*.
  - **External reference** — another company's design, competitor UI, or an
    inspiration source.
  If the classification is ambiguous, ask the user.
- **Text description** — the vibe, product, audience, references ("playful
  coffee shop app, warm colors, rounded corners").

#### Handling each medium

| Medium | How to process |
| --- | --- |
| Image (screenshot, mockup, logo, brand sheet) | Read with the file-read tool and analyze visually. |
| Video | If the agent reads video natively, watch it. Otherwise extract representative frames with `ffmpeg` and analyze them as images — scene detection picks distinct shots: `ffmpeg -i in.mp4 -vf "select='gt(scene,0.3)'" -vsync vfr frame_%03d.png`; fixed sampling also works (`fps=1/2` = one frame every 2s). Note motion and transition language in the prose rationale. |
| Document (PDF, markdown, text) | Read directly — PDFs usually carry both text and embedded images. Brand guidelines are authoritative for exact values. |
| Office files (.docx/.pptx/.key) | They are zip archives: extract text (pandoc if available, else the document XML) and pull embedded media (`unzip -j file.pptx 'ppt/media/*' -d media/`), then analyze extracted images as images. |
| URL | Classify first. For **this project's** site, prefer tokens from source over pixels: fetch the page and its stylesheets (`curl`), parse hex/rgb/hsl colors, font-family stacks, radius and spacing values; optionally capture screenshots with a headless browser (puppeteer/playwright/`chrome --headless --screenshot`) at desktop and mobile widths and analyze them as images. For **someone else's** site, treat the URL as an external reference — patterns only, no tokens. |

### 2. Decide the source of truth

| Situation | Anchors the tokens | Role of the other inputs |
| --- | --- | --- |
| Codebase has an established design | The codebase (existing tokens, fonts, component styling) | Project artifacts extend or validate it; external references inform patterns only; text refines naming and rationale |
| No codebase design + project artifact media | The artifact (sample values faithfully) | Text shapes intent, naming, and rationale |
| Text only | The text, translated into concrete values with discipline | The codebase scan confirms there is nothing to preserve |

If there is **no** codebase identity, no media, and no text, ask before
inventing anything.

### 3. Analyze

From **project artifacts** (screenshots, video frames, captured pages,
guideline documents), extract:

- **Palette** — background/surface colors, primary text color, accent/CTA
  color, secondary accents, error red if visible. Sample hex as close to the
  source as possible for flat UI captures. Do not invent colors that are
  not in the artifact or the codebase.
- **Typography** — serif/sans/mono character, weight contrast, size hierarchy,
  uppercase/caps usage. Name a concrete font family when the
  source makes it recognizable; otherwise pick one that matches the character.
- **Shapes** — corner radius family (sharp 0–4px, rounded 8–12px, pill
  9999px), border vs shadow usage.
- **Layout & spacing** — grid feel, whitespace generosity, likely base unit
  (4px or 8px).
- **Components** — button fills/outlines, input borders, card treatment,
  navigation pattern.

From **external references**, extract only transferable qualities — never
tokens (see *External references: adapt, don't adopt* below): layout
structure, density, spacing rhythm, shape language, interaction patterns, and
overall atmosphere.

From **text only**, translate adjectives into concrete values with discipline:

- Accent saturation ≤ 80% — no neon.
- No pure black `#000000` — use off-blacks (e.g. `#18181B`).
- Maximum one accent color.
- Every text/background pair must pass WCAG AA (4.5:1).

#### External references: adapt, don't adopt

When the user attaches a design that belongs to another company or product:

- Do **not** copy its brand tokens (exact palette, fonts) into the project's
  `DESIGN.md` unless the user explicitly asks to adopt that identity — and
  even then confirm first, because the project may not have the right to use
  it.
- Translate what makes the reference work into the project's own language:
  re-express its density, rhythm, and shape language through the codebase's
  existing palette and typography.
- If no project identity exists yet, derive an original, coherent system
  *inspired by* the reference (same feel, different expression) and say so in
  the report.
- Naming and prose rationale should describe the project's product, never the
  external brand.

### 4. Write DESIGN.md

Default output path: `DESIGN.md` at the project root.

If `DESIGN.md` already exists, do **not** silently overwrite: summarize what
would change and confirm. When regenerating, preserve unknown/custom sections
and tokens that are not derivable from the inputs.

Write the front matter and body exactly per the format below. Front matter must
open and close with a line containing exactly `---`.

Minimal example:

```markdown
---
version: alpha
name: DevFocus Dark
colors:
  primary: "#2665fd"
  secondary: "#475569"
  surface: "#0b1326"
  on-surface: "#dae2fd"
  error: "#ffb4ab"
typography:
  body-md:
    fontFamily: Inter
    fontSize: 16px
    fontWeight: 400
    lineHeight: 1.6
  label-md:
    fontFamily: Inter
    fontSize: 12px
    fontWeight: 500
    letterSpacing: 0.1em
rounded:
  md: 8px
spacing:
  sm: 8px
  md: 16px
components:
  button-primary:
    backgroundColor: "{colors.primary}"
    textColor: "#ffffff"
    rounded: "{rounded.md}"
    padding: 12px
---

# Design System

## Overview

A focused, minimal dark interface for a developer productivity tool.
Clean lines, low visual noise, high information density.

## Colors

- **Primary** (#2665fd): CTAs, active states, key interactive elements
- **Surface** (#0b1326): page backgrounds
- **On-surface** (#dae2fd): primary text on dark backgrounds

## Typography

- **Headlines**: Inter, semi-bold
- **Body**: Inter, regular, 16px
- **Labels**: Inter, medium, 12px, uppercase for section headers

## Layout

Single-column layout, max content width 960px, 8px spacing base.

## Elevation & Depth

Flat design: depth comes from borders and background contrast, not shadows.

## Shapes

All interactive elements use an 8px corner radius. Never mix rounded and
sharp corners in the same view.

## Components

- **Buttons**: 8px radius, primary uses brand blue fill
- **Inputs**: 1px border, subtle surface-variant background
- **Cards**: no elevation, 1px outline border

## Do's and Don'ts

- Do use the primary color sparingly, only for the most important action
- Don't mix rounded and sharp corners in the same view
- Do maintain 4.5:1 contrast ratio for all text
```

#### Front matter schema

```yaml
version: alpha            # optional; current version is "alpha"
name: <string>            # required; display name of the design system
description: <string>     # optional
colors:
  <token-name>: <Color>                 # "#RRGGBB" (sRGB hex)
typography:
  <token-name>: <Typography>            # composite object, see below
rounded:
  <scale-level>: <Dimension>            # e.g. sm: 4px
spacing:
  <scale-level>: <Dimension | number>   # e.g. md: 16px
components:
  <component-name>:
    <token-name>: <string | {token.reference}>
```

Token types:

| Type | Format | Example |
| --- | --- | --- |
| Color | `#` + hex (sRGB) | `"#1A1C1E"` |
| Dimension | number + unit (px, em, rem) | `48px`, `-0.02em` |
| Token reference | `{path.to.token}` | `{colors.primary}` |
| Typography | composite object | see properties below |

Typography properties: `fontFamily` (string), `fontSize` (Dimension),
`fontWeight` (number; bare numbers and quoted strings are equivalent),
`lineHeight` (Dimension or unitless multiplier — unitless recommended),
`letterSpacing` (Dimension), `fontFeature` (font-feature-settings),
`fontVariation` (font-variation-settings).

Token references point at a **primitive** value (`{colors.primary-60}`, not
`{colors}`). Only inside `components` may a reference target a composite value
(`typography: "{typography.label-md}"`).

Recommended (not required) token names:

- colors: `primary`, `secondary`, `tertiary`, `neutral`, `surface`,
  `on-surface`, `error`
- typography: `headline-display`, `headline-lg`, `headline-md`, `body-lg`,
  `body-md`, `body-sm`, `label-lg`, `label-md`, `label-sm`
- rounded: `none`, `sm`, `md`, `lg`, `xl`, `full`

Component sub-token names (recognized set): `backgroundColor`, `textColor`,
`typography`, `rounded`, `padding`, `size`, `height`, `width`. Variants are
sibling keys with a related name (`button-primary` / `button-primary-hover`).

#### Body section order (canonical)

Sections use `##` headings. An optional single `#` title may appear but is not
a section. Omit sections that do not apply; never reorder the ones present;
never duplicate a `##` heading. Unknown sections are allowed and must be
preserved by consumers — extending is encouraged.

| # | Section | Aliases | Content |
| --- | --- | --- | --- |
| 1 | Overview | Brand & Style | Brand personality, audience, emotional response; the fallback context when a token is undefined |
| 2 | Colors | | Palettes; at least `primary`. Map descriptive names ("Midnight Forest") to token names (`primary`) |
| 3 | Typography | | Levels with semantic role + size variant (most systems have 9–15) |
| 4 | Layout | Layout & Spacing | Grid model, spacing scale, containment principles |
| 5 | Elevation & Depth | Elevation | Shadow properties, or the flat alternative (borders, tonal layers, contrast) |
| 6 | Shapes | | Corner radii, edge treatments, shape language |
| 7 | Components | | Buttons, Chips, Lists, Inputs, Checkboxes, Radio buttons, Tooltips + domain-specific ones |
| 8 | Do's and Don'ts | | Actionable guardrails and common pitfalls |

Write prose in the project's primary language (match the README); token keys
and section headings stay in English.

### 5. Validate (manual checklist)

Run through every item before declaring done. These mirror the 8 rules of the
DESIGN.md linter — fix findings, do not ship around them.

1. **Delimiters** — front matter opens and closes with a line containing
   exactly `---`.
2. **broken-ref (error)** — every `{token.reference}` resolves to a token
   defined in the front matter; every component property is in the recognized
   sub-token set.
3. **missing-primary (warning)** — `colors.primary` is defined.
4. **missing-typography (warning)** — at least one typography token exists.
5. **missing-sections (info)** — `spacing` and `rounded` are defined (or their
   absence is a conscious, user-approved choice).
6. **section-order (warning)** — body sections follow the canonical order; no
   duplicate `##` headings.
7. **orphaned-tokens (warning)** — when components exist, every color token is
   referenced by at least one component property (or remove the token).
8. **contrast-ratio (warning)** — every component `backgroundColor` /
   `textColor` pair is ≥ 4.5:1.

Contrast math (WCAG relative luminance):

```
L = 0.2126*R + 0.7152*G + 0.0722*B
channel c: c = c/255; c = c <= 0.03928 ? c/12.92 : ((c+0.055)/1.055)^2.4
ratio = (L_light + 0.05) / (L_dark + 0.05)   # must be >= 4.5
```

Quality bar on top of the lint rules:

- Every token in the front matter is mentioned or used in the prose (bimodal:
  agents read tokens, humans read rationale).
- Do's and Don'ts are specific and enforceable, not platitudes.
- No invented brand facts (no fake metrics, no made-up logo usage rules).

### 6. Wire the references

Add a short Design System section pointing at `DESIGN.md`. Always
**idempotent**: if the target file already mentions `DESIGN.md`, skip it and
say so.

| File | Exists | Action |
| --- | --- | --- |
| `README.md` | yes | Append the README snippet below |
| `README.md` | no | Skip (do not create a README just for this) |
| `AGENTS.md` | yes | Append the agent snippet below |
| `AGENTS.md` | no | Create minimal file containing the agent snippet |
| `CLAUDE.md` | yes | Append the agent snippet below (skip if it is a symlink to `AGENTS.md`) |
| `CLAUDE.md` | no | Create minimal file containing the agent snippet |
| Other agent entrypoints that already exist (e.g. `GEMINI.md`, `.cursorrules`, `.github/copilot-instructions.md`) | yes | Append the agent snippet below |
| Same, but missing | no | Leave alone; mention them in the summary |

README snippet:

```markdown
## Design System

This project's visual identity — colors, typography, spacing, and component
rules — is defined in [DESIGN.md](./DESIGN.md). Read it before generating or
modifying any UI.
```

Agent snippet:

```markdown
## Design System

Before generating or modifying any UI, read [DESIGN.md](./DESIGN.md) and
follow its design tokens (colors, typography, spacing, rounded) and component
rules exactly.
```

When creating a minimal `AGENTS.md` / `CLAUDE.md`, use this skeleton so the
file is valid on its own:

```markdown
# AGENTS.md

## Design System

Before generating or modifying any UI, read [DESIGN.md](./DESIGN.md) and
follow its design tokens (colors, typography, spacing, rounded) and component
rules exactly.
```

(`# CLAUDE.md` for the Claude file.)

If the reference path is not the project root (e.g. `docs/DESIGN.md`), use
that relative path in every snippet.

### 7. Report

Finish with a short summary: where `DESIGN.md` was written, the token counts
per section (colors / typography / rounded / spacing / components), which
files were updated or created, any checklist items consciously skipped, and —
when external references were attached — how each was adapted (which
qualities were transferred and which brand tokens were deliberately left
out).

## Gotchas

- **Tokens only in prose are not enforced.** Agents key off the YAML front
  matter; anything that must be exact (hex, px, weights) belongs there too.
- **One accent color.** Multiple competing accents is the fastest route to a
  generic, noisy look.
- **Don't round sampled hex values** to the nearest Tailwind palette entry
  when the source capture has exact flat colors — fidelity beats tidy values.
- **Aliases count for ordering**: a section titled "Elevation" is
  "Elevation & Depth"; "Brand & Style" is "Overview".
- **Unknown content is an extension, not an error** — but a duplicated
  `## Colors` heading is invalid; merge instead.
- **Symlinked CLAUDE.md → AGENTS.md** is a common setup; editing both would
  duplicate the section. Check before writing.
- **An external reference is inspiration, not a brand to clone.** Copying
  another company's exact palette or fonts into `DESIGN.md` couples the
  project to an identity it does not own; transfer structure and rhythm, not
  tokens.
- The DESIGN.md is a living artifact: re-run this skill when the design
  evolves rather than hand-editing tokens in five places.
