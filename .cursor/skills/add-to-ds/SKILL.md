---
name: add-to-ds
description: Add any artifact to the design system — primitive component, section, or composition page. Determines type automatically from the Figma URL. Strictly uses only existing components from src/components/ — never invents or approximates. If a required component or section is missing, stops and asks for its Figma link before continuing.
---

# Add to Design System

Single entry point for adding any artifact to the DS. Takes a Figma URL, determines what it is, builds it correctly — or stops and asks for what's missing before proceeding.

**Core rule: no improvisation.** Every piece of UI must come from Figma or from an existing component in `src/components/`. If something is needed but missing — the agent stops and asks. It never invents layout, never approximates a component, never guesses at styles.

---

## Required Input

A Figma URL pointing to the artifact to build.

**If no URL was provided** — ask:

> Please share a Figma link to what you want to build.

---

## Step 1: Identify artifact type

Call `get_metadata(fileKey, nodeId)` on the provided URL. Analyse the structure:

**Primitive component** — contains `symbol` children with variant names (e.g. `Variant=Primary, State=Default`). No `instance` children referencing other DS components. Atomic level: Button, Input, Badge, Avatar, Tag, etc.

**Section** — contains `instance` children referencing named DS components. A layout block assembled from primitives: Header, Footer, Hero, ProductGrid, etc.

**Composition page** — contains multiple section-level instances assembled into a full page (Header + content + Footer), or the user explicitly says "page" / "full page".

**If the node contains multiple distinct component frames** — list them and ask:

> I found these in the Figma link:
> - [Name] — [primitive / section]
> - [Name] — [primitive / section]
>
> Which should I build? Or all of them?

Wait for the answer, then proceed one by one.

---

## Step 1a: Check if artifact already exists

After identifying the type and name in Step 1, check if the artifact already exists in the project:

- **Primitive** → check `src/components/[Name]/[Name].tsx`
- **Section** → check `src/sections/[Name]/[Name].tsx`
- **Composition page** → check `sandbox/pages/[Name].tsx`

**If the file does not exist** → proceed to Step 1b (new artifact).

**If the file exists** → switch to update mode.

---

### Update mode

Figma is the source of truth. The existing code is treated as the current implementation — Figma defines what it should be.

#### U1 — Fetch current state from Figma

Call `get_design_context(fileKey, nodeId)` and `get_variable_defs(fileKey, nodeId)`.

Read the existing `.tsx` and `.css` files in full.

#### U2 — Build a diff

Compare Figma output against existing code across these dimensions:

| Dimension | What to check |
|---|---|
| **Props / variants** | New variants or states in Figma that don't exist in code? Props in code that no longer exist in Figma? |
| **Tokens** | Token names changed? New tokens used? Hardcoded values that should be tokens? |
| **Layout** | Spacing, sizing, flex rules changed? |
| **Structure** | New elements added or removed? Order changed? |
| **Typography** | Font size, weight, line-height changed? |

#### U3 — Classify each change

For every difference found, assign a category:

- **Safe** — purely additive or visually obvious (new variant, token value change, spacing tweak). Apply automatically.
- **Breaking** — removes or renames an existing prop, changes component API, affects how the component is used elsewhere. Report before applying.
- **Ambiguous** — agent is not sure whether the Figma change is intentional or if the code diverged for a reason (e.g. a workaround, a local fix, a prop added manually). Report and ask.

#### U4 — Report before writing

Before changing anything, present a summary:

> **Updating [Name]** — here's what I found:
>
> ✅ **Safe changes (will apply automatically):**
> - Token `--old-token` → `--new-token` on hover background
> - Added `size="large"` variant — new in Figma
> - Spacing on padding changed from `space-200` to `space-300`
>
> ⚠️ **Breaking changes (need your confirmation):**
> - Prop `disabled` renamed to `state="disabled"` in Figma — this will break existing usages
> - Variant `subtle` removed from Figma — delete from code?
>
> ❓ **Ambiguous (need clarification):**
> - Code has `scheme="danger"` prop that doesn't exist in this Figma node — was this added intentionally or should it be removed?
> - Figma shows `border-radius: 12px` but code uses `var(--token-radius-200, 8px)` — Figma value not mapped to a token. Should I use the token closest in value, or hardcode?
>
> Shall I proceed with safe changes now and wait for your answers on the rest?

Wait for the user's response. Then:
- Apply safe changes immediately
- Apply breaking/ambiguous changes only after explicit confirmation for each
- If user says "apply all" — apply everything and note what was changed

#### U5 — Apply changes with str_replace

Use `str_replace` for surgical edits — never rewrite the whole file unless the structure changed so fundamentally that a full rewrite is the only clean option. If a full rewrite is needed — say so before doing it.

#### U6 — Validate

```bash
npm run ds:check
```

Report what was updated, what was skipped, and any open questions that weren't resolved.

---

## Step 1b: Assess responsive layout

Before routing, check whether the artifact requires responsive behaviour and whether there's enough information to implement it correctly.

This step applies to **sections and composition pages** only. Primitives don't need responsive assessment.

### What to look for in Figma

Call `get_metadata` on the parent of the provided node. Look for sibling frames or named variants that suggest different viewports:

**Clear responsive signal** — multiple frames with meaningfully different widths:
- Names like Desktop / Tablet / Mobile, or Wide / Narrow, or 1440 / 768 / 375
- Same content, different layout or column count
- `get_metadata` shows siblings with widths like 1200px, 768px, 375px

**Ambiguous signal** — only one frame exists, but:
- The layout has multiple columns that would need to reflow on mobile
- There are grid areas that clearly need to change at smaller sizes
- The section is wide enough that it would break on mobile without CSS changes

**No responsive needed** — single-column layout, full-width, no columns to reflow.

---

### Decision tree

**If clear responsive frames exist in Figma:**

- Use `get_metadata` on the **parent** of the provided node to find all sibling frames
- Sort siblings by width descending: widest = Desktop baseline, others = breakpoints
- Call `get_design_context` on each frame separately
- Derive breakpoint values: `breakpoint = next_larger_frame_width - 1px`
  - Example: Desktop = 1200px, Tablet = 768px → `@media (max-width: 1199px)`
  - Example: Tablet = 768px, Mobile = 375px → `@media (max-width: 767px)`

**Visibility diff — run this for every element across all frames:**

For each named element found in the Desktop frame, check its status in every smaller frame using `get_metadata`:

| Element in Desktop | Element in smaller frame | Result |
|---|---|---|
| visible (no `hidden`) | `hidden="true"` | `display: none` inside that `@media` block |
| visible | absent entirely | `display: none` inside that `@media` block |
| `hidden="true"` | visible (no `hidden`) | `display: none` on desktop baseline, show inside `@media` |
| `hidden="true"` | absent | hidden everywhere — omit from render or render with `display: none` unconditionally |

**Translate visibility to CSS:**

```css
/* Desktop baseline */
.navList     { display: flex; }   /* visible on desktop */
.burgerBtn   { display: none; }   /* hidden on desktop */

@media (max-width: 767px) {
  .navList   { display: none; }   /* hidden on mobile */
  .burgerBtn { display: flex; }   /* visible on mobile */
}
```

**Interactive states (open/closed menu, expanded/collapsed):**

If Figma has separate frames for open and closed states of the same element (e.g. `Header/Mobile/MenuOpen` and `Header/Mobile/MenuClosed`):
- Implement with `useState` in React
- CSS shows/hides based on the state class or inline style
- Use the open-state frame to get the exact styles for the expanded element

If Figma only has one state but the element clearly needs interactivity (a hamburger button with no open state drawn) — stop and ask:

> The mobile menu button exists in the design but I don't see a drawn open/expanded state. Should I:
> 1. Ask you to draw the open state in Figma so I can implement it exactly
> 2. Make a reasonable assumption (slide-in overlay, full-width nav list) — at your own risk

**If `src/layout.css` exists:**

- Read it to extract existing breakpoint values and column classes
- Use those breakpoints and `.l-container`, `.l-col-*` classes where appropriate
- Proceed — no need to ask

**If `src/layout.css` does not exist:**

- Do NOT create it and do NOT ask the user to create it
- Write responsive CSS directly in the section's own `.css` file — this is perfectly valid
- Each section owns its own `@media` queries
- `src/layout.css` is optional infrastructure — it only makes sense to create it (via `init-layout` skill) when multiple sections share the same grid rules. That's a separate decision, not required by this skill.

**If only one frame exists AND layout clearly needs responsive behaviour:**

Stop and ask — three options presented to the user:

> I only have the desktop version of **[Name]**. For mobile and tablet I can:
>
> 1. **Ask you** — share Figma links to the tablet and/or mobile frames and I'll implement them exactly
> 2. **Use layout rules** — share a link to your Composition Guide / grid rules page in Figma and I'll derive breakpoints from it
> 3. **My best guess** — I'll make reasonable responsive assumptions (reflow columns to single column on mobile, adjust spacing). Fast, but not guaranteed to match your design intent. Proceed at your own risk.
>
> Which do you prefer?

Wait for the answer:
- **Option 1** — user shares frame links → call `get_design_context` on each → proceed with full responsive CSS
- **Option 2** — user shares Composition Guide link → call `get_metadata` + `get_design_context` on viewport frames → derive breakpoints → proceed
- **Option 3** — apply standard assumptions, add a comment in CSS marking them as estimates:

```css
/* ⚠️ Responsive breakpoints below are best-guess estimates.
   No tablet/mobile frames were provided in Figma.
   Review and adjust to match your design intent. */
```

**If the layout is clearly single-column or full-width:**

No responsive handling needed — proceed without asking.

---

## Tailwind → CSS Modules conversion

`get_design_context` returns Tailwind utility classes. The project uses CSS Modules. **Always convert before writing any code.** Never copy Tailwind classes into `.tsx` or `.css` files.

### Layout

| Tailwind from Figma | CSS Modules |
|---|---|
| `w-[var(--token,1200px)]` on a section root | `width: 100%; max-width: 1200px; margin: 0 auto;` — never use fixed `width` on a container |
| `w-full` / `size-full` | `width: 100%` |
| `flex` | `display: flex` |
| `flex-col` | `flex-direction: column` |
| `flex-wrap` | `flex-wrap: wrap` |
| `flex-[1_0_0]` / `flex-1` | `flex: 1 1 0; min-width: 0;` |
| `shrink-0` | `flex-shrink: 0` |
| `items-center` | `align-items: center` |
| `items-start` | `align-items: flex-start` |
| `justify-center` | `justify-content: center` |
| `justify-end` | `justify-content: flex-end` |
| `justify-between` | `justify-content: space-between` |
| `content-stretch` | `align-content: stretch` |
| `content-center` | `align-content: center` |
| `relative` | `position: relative` |
| `absolute` | `position: absolute` |
| `overflow-clip` / `overflow-hidden` | `overflow: hidden` |
| `min-h-px` | `min-height: 1px` |
| `min-w-px` | `min-width: 1px` |
| `min-w-0` | `min-width: 0` |
| `inset-[...]` | `inset: ...` |

### Spacing

| Tailwind from Figma | CSS Modules |
|---|---|
| `p-[var(--token,32px)]` | `padding: var(--token, 32px)` |
| `px-[var(--token,16px)]` | `padding-left: var(--token, 16px); padding-right: var(--token, 16px)` |
| `py-[var(--token,16px)]` | `padding-top: var(--token, 16px); padding-bottom: var(--token, 16px)` |
| `gap-[var(--token,24px)]` | `gap: var(--token, 24px)` |
| `gap-[var(--token,0px_24px)]` | `gap: 0 var(--token, 24px)` — split `row_col` notation into two values |
| `gap-x-[...]` | `column-gap: ...` |
| `gap-y-[...]` | `row-gap: ...` |

> **`0px_24px` notation:** Figma uses underscore to separate row-gap and column-gap. `gap-[var(--token,0px_24px)]` means row-gap=0, column-gap=24px → `gap: 0 var(--token, 24px)`. Never copy the underscore into CSS — it produces invalid output.

### Sizing

| Tailwind from Figma | CSS Modules |
|---|---|
| `w-[40px]` | `width: 40px` |
| `h-[35px]` | `height: 35px` |
| `size-[16px]` | `width: 16px; height: 16px` |
| `max-w-none` | `max-width: none` |
| `w-[var(--sds-responsive-device-width,1200px)]` | `width: 100%; max-width: 1200px; margin: 0 auto` — treat as container, not fixed width |

### Colors & borders

| Tailwind from Figma | CSS Modules |
|---|---|
| `bg-[var(--token,#fff)]` | `background: var(--token, #fff)` |
| `text-[color:var(--token,#000)]` | `color: var(--token, #000)` |
| `border-[var(--token,#ccc)]` | `border-color: var(--token, #ccc)` |
| `border-solid` | `border-style: solid` |
| `border-[length:var(--token,1px)]` | `border-width: var(--token, 1px)` |
| `border-b-[length:var(--token,1px)]` | `border-bottom-width: var(--token, 1px)` |
| `rounded-[var(--token,8px)]` | `border-radius: var(--token, 8px)` |

### Typography

| Tailwind from Figma | CSS Modules |
|---|---|
| `font-[family-name:var(--token,'Inter:Regular',sans-serif)]` | `font-family: var(--token, 'Inter', sans-serif)` — strip the `family-name:` prefix and the `:Regular` style suffix from the fallback |
| `font-[var(--token,400)]` | `font-weight: var(--token, 400)` |
| `text-[length:var(--token,16px)]` | `font-size: var(--token, 16px)` |
| `leading-none` | `line-height: 1` |
| `leading-[1.4]` | `line-height: 1.4` |
| `not-italic` | `font-style: normal` |
| `whitespace-nowrap` | `white-space: nowrap` |

### What to ignore

These Tailwind classes have no CSS Modules equivalent — omit them:

- `content-stretch`, `content-start`, `content-center` — handled by flexbox context
- `relative`, `shrink-0` — include only when they affect visible layout
- `not-italic` — omit unless italic is inherited from parent
- All `data-node-id` attributes — debug only, strip from final output

---

## Icons

`get_design_context` exports icons as **temporary PNG URLs** (`https://www.figma.com/api/mcp/asset/...`) that expire in 7 days. SVG source is not available through MCP — only through a direct Figma link to the icon node.

**The agent never draws or generates icons itself. Every icon must come from Figma.**

---

### Step I1 — Detect icons in the output

Scan the `get_design_context` response for:
- `const imgIcon... = "https://www.figma.com/api/mcp/asset/..."` constants
- `<img src={imgIcon...} />` elements
- `<div data-name="Icon">` or `<div data-name="[icon-name]">` wrapping an `<img>`

If none found — no icons, proceed normally.

If icons found — collect their names from `data-name` attributes (e.g. `Star`, `X`, `Arrow`, `Check`) and go to Step I2.

---

### Step I2 — Check the project icon library

Check `src/icons/` for existing `.svg` files. Match by name (case-insensitive, ignore separators: `star.svg`, `Star.svg`, `icon-star.svg` all match `data-name="Star"`).

**For each icon from Step I1:**

| Case | Action |
|---|---|
| SVG file found in `src/icons/` | Use it — import as React component via `?react` |
| SVG file not found | Stop — ask for Figma link |

---

### Step I3 — Missing icons: ask for SVG file upload

If any icon is missing from `src/icons/` — stop immediately. Do not proceed with building the component. Do not generate, draw, or approximate any icon.

> The following icons are needed but not found in `src/icons/`:
>
> - **Star** — used in Button (icon start slot)
> - **X** — used in Button (icon end slot)
>
> Please upload the SVG files for each icon and I'll add them to `src/icons/`.

Wait. Do not continue until the user uploads the files.

When the user uploads SVG files:

1. Read each uploaded file — verify it starts with `<svg`
2. If the file is not valid SVG — ask the user to re-upload:
   > `[filename]` doesn't look like a valid SVG file. Please upload the correct file.
3. Clean up the SVG before saving:
   - Remove all hardcoded `fill` and `stroke` color attributes from every element — replace with `currentColor` so the icon inherits color from CSS tokens
   - Remove `width` and `height` attributes from the root `<svg>` element — size is always controlled by CSS
   - Keep `viewBox` — it is required for correct scaling
4. Save to `src/icons/[name].svg` — use the icon name from `data-name` in Figma (lowercased, spaces → hyphens)
5. Confirm and continue:
   > Saved `src/icons/star.svg` and `src/icons/x.svg`. Continuing with the component.

**Never:**
- Generate or draw SVG paths from scratch
- Guess at icon shapes
- Use the temporary Figma MCP PNG URLs (`figma.com/api/mcp/asset/...`) in any code
- Use `data:image/svg+xml,...` strings as JSX component names — React will throw `InvalidCharacterError: tag name is not a valid name`
- Use `<img src={dataUriString} />` for icons — always import as React component
- Proceed without the actual SVG files

---

### Special case: user uploads SVG files directly

If the user uploads one or more `.svg` files without any other context — treat it as an icon library update:

1. For each uploaded file:
   - Verify it's valid SVG (starts with `<svg`)
   - Apply cleanup: remove hardcoded `fill`/`stroke`, remove root `width`/`height`, keep `viewBox`
   - Check if `src/icons/[name].svg` already exists
     - Exists → overwrite and mark as updated
     - Doesn't exist → save as new
2. Report:
   > Icon library updated:
   > - Added: star.svg, arrow-right.svg, check.svg
   > - Updated: x.svg
   > - Invalid (skipped): broken.svg — not a valid SVG file

---

### Step I4 — Use icons in components

**Import pattern (Vite):**
```tsx
import StarIcon from "../../icons/star.svg?react";
import XIcon from "../../icons/x.svg?react";
```

**Usage:**
```tsx
{iconStart && <StarIcon className={styles.icon} />}
{iconEnd && <XIcon className={styles.icon} />}
```

**CSS:**
```css
.icon {
  width: 16px;
  height: 16px;
  color: inherit;
  flex-shrink: 0;
  display: block;
}
```

**Rules:**
- Always `color: inherit` — never hardcode icon color
- Never `<img src="...figma.com/api/mcp/asset/...">` in component code
- Never inline SVG path data in TSX — always import from `src/icons/`
- Never generate or approximate SVG paths — every icon comes from an uploaded file

---

## Step 2: Route by type

---

### Route A — Primitive component

**Output:** `src/components/[Name]/[Name].tsx` + `[Name].css`

#### A1 — Detect mergeability

If the page contains multiple frames that are structurally identical (same layout, same props, only token values differ — like Button and Button Danger):
- **One file** with an extra prop (`scheme`, `intent`, or `tone`)
- Genuinely different structure or API → separate files

#### A2 — Fetch design context

Call `get_design_context(fileKey, nodeId)` on the component frame.
Call `get_variable_defs(fileKey, nodeId)` to get exact CSS variable names.

#### A3 — Read token prefix

Open `tokens/tokens.css`. Extract the prefix from the first `--*` declaration.

**If `tokens/tokens.css` is missing — stop:**

> `tokens/tokens.css` not found. Run `build-tokens-css` first.

#### A4 — Build component files

**`[Name].tsx`:**
- `import React from "react"` — always first, never omit
- CSS import: `import styles from './[Name].css'` — never `import [Name] from './[Name].css'` (CSS has no default export)
- Named union types for each prop dimension (variant, state, size, scheme)
- Props extend `React.HTMLAttributes` of root element, `Omit` conflicting props
- Local `cls()` helper for conditional class joining — no external lib
- `className` prop + `...rest` spread on root
- Default prop values in destructuring
- Named export + `export default`

**`[Name].css`:**
- File extension: always `.css` — never `.module.css` unless the entire project consistently uses CSS Modules with `.module.css` naming (check existing components first)
- Comments: always `/* comment */` — never `//` (CSS does not support `//` comments, they cause PostCSS parse errors)
- Every color, spacing, radius, typography value → `var(--[prefix]-*)` from `tokens/tokens.css`
- No raw `px` for spacing/radius, no `#hex`, no `rgb()`, no `hsl()`
- Full variant × state matrix: `.buttonVariantPrimary.buttonStateHover { ... }`
- `transition` for interactive state changes
- No fixed `width`/`max-width`, no outer `margin`

**If a required token is missing from `tokens/tokens.css` — stop:**

> Missing token for `[property]` — the design uses `[value]` but it's not in `tokens/tokens.css`.
>
> Options:
> 1. Re-run `build-tokens-css` (token may exist in Figma but wasn't exported)
> 2. Add the token manually to `tokens/tokens.css`
> 3. Confirm a hardcoded fallback is acceptable for now

Wait for the user's decision. Do not write hardcoded values without explicit confirmation.

#### A5 — Validate

```bash
npm run ds:check
```

Fix any reported hardcoded values before finishing.

---

### Route B — Section

**Output:** `src/sections/[Name]/[Name].tsx` + `[Name].css`

#### B1 — Inventory required components

Call `get_metadata(fileKey, nodeId)`. Collect all `instance` names — these are the components the section depends on.

#### B2 — Check dependencies

For each required component, check `src/components/[ComponentName]/[ComponentName].tsx`.

**If anything is missing — stop immediately:**

> To build **[SectionName]** I need these components which don't exist yet:
>
> | Component | Status |
> |---|---|
> | NavigationPill | ❌ missing |
> | IconButton | ❌ missing |
>
> Please share a Figma link for each missing component and I'll build them first.

Wait. For each link provided — run **Route A** in full. Then resume from B1.

**Do not proceed until all dependencies exist. No placeholders. No approximations.**

#### B3 — Fetch design context

Call `get_design_context(fileKey, nodeId)` on the section node. Use the responsive strategy already determined in Step 1b — if additional frames were provided there, fetch them here too.

#### B4 — Build section files

**`[Name].tsx`:**
- `import React from "react"`
- CSS import: `import styles from './[Name].css'` — never `import [Name] from './[Name].css'`
- Import only from `src/components/` — no inline primitive logic inside the section
- Content passed as props (text, items, links) — no hardcoded copy
- If `src/layout.css` exists — use its classes for container/column layout; otherwise write layout directly in the section's CSS

**`[Name].css`:**
- File extension: always `.css` — never `.module.css` unless all existing sections use that naming
- Comments: always `/* comment */` — never `//`
- All values via `var(--[prefix]-*)` tokens — no hardcoded values
- Apply responsive CSS as determined in Step 1b (exact breakpoints from Figma frames, from `layout.css`, best-guess, or none)
- No outer `margin` on root — parent controls spacing

**If a required token is missing** — same stop-and-report as Route A.

#### B5 — Validate

```bash
npm run ds:check
```

---

### Route C — Composition page

**Output:** `sandbox/pages/[Name].tsx` + `sandbox/pages/[Name].html`, entry in `sandbox/pages/index.tsx`

#### C1 — Inventory required sections

Call `get_metadata(fileKey, nodeId)`. Collect all top-level section instances.

**If anything is missing — stop immediately:**

> To build **[PageName]** I need these sections which don't exist yet:
>
> | Section | Status |
> |---|---|
> | HeroBasic | ❌ missing |
> | ProductGrid | ❌ missing |
>
> Please share a Figma link for each missing section and I'll build them first.

Wait. For each link — run **Route B** in full (which may trigger **Route A** for missing components). Then resume from C1.

#### C2 — Build page files

**`sandbox/pages/[Name].html`:**
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>[Page Name]</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="./[Name].tsx"></script>
  </body>
</html>
```

**`sandbox/pages/[Name].tsx`:**
```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import "../../../tokens/tokens.css";
import "../../sandbox.css";
// import "../../../src/layout.css"; // uncomment if layout.css exists in the project
import { SectionA } from "../../../src/sections/SectionA/SectionA";
// ...sections in page order

function App() {
  return (
    <>
      <SectionA />
      {/* sections in order from Figma */}
    </>
  );
}

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode><App /></React.StrictMode>
);
```

Rules:
- Import only from `src/sections/` — no raw components directly on the page
- If `src/layout.css` exists — import it and use its grid classes; otherwise sections handle their own layout
- No sandbox wrapper div
- Pass realistic placeholder content as props where sections accept it

#### C3 — Register in pages index

Read `sandbox/pages/index.tsx`. Use `str_replace` to add to the `pages` array:

```tsx
{ name: "[Name]", description: "[what sections this page contains, one sentence]", href: "./[Name].html" },
```

#### C4 — Open in browser

Check if port 5173 is in use (`lsof -ti:5173`):
- **Running** → open `http://localhost:5173/sandbox/pages/[Name].html` in Cursor Simple Browser. Do NOT run `npm run sandbox:dev`.
- **Not running** → run `npm run sandbox:dev`, wait for start, then open the URL.

Report:
> Page ready: http://localhost:5173/sandbox/pages/[Name].html
> Index: http://localhost:5173/sandbox/pages/index.html

---

## Dependency chain

```
Composition page
  └─ needs Sections     → missing? ask for Figma link → Route B
       └─ needs Components → missing? ask for Figma link → Route A
            └─ needs Tokens  → missing? stop → run build-tokens-css
```

At every level: ask for the Figma link, build it correctly, then continue up. Never skip, never approximate, never invent.