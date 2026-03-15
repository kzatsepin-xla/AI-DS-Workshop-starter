---
name: add-to-sandbox
description: Add any design system artifact to the sandbox. Input is a folder path with a .tsx + .css pair. Determines type automatically (primitive / section / composition page), creates the sandbox entry point if missing, then adds the demo. Never touches src/ or tokens/.
---

# Add to Sandbox

Takes a folder path, determines what type of artifact it is, and adds a demo to the correct sandbox entry point.

If the sandbox entry file doesn't exist yet — creates it first, then adds the artifact as the first item.
If it already exists — adds the artifact surgically without touching anything else.

## Required Input

A folder path containing a `.tsx` + `.css` pair:

```
src/components/Button
src/sections/Header
src/sections/HeroBasic
```

**If no path was provided** — ask:

> What's the folder path? (e.g. `src/components/Button` or `src/sections/Header`)

---

## Step 1: Read the source folder

Read both files:
- `[folder]/[Name].tsx`
- `[folder]/[Name].css`

If either is missing — stop:

> `[Name].tsx` or `[Name].css` not found in `[folder]`. Make sure the folder exists and contains both files.

---

## Step 2: Determine artifact type

Use the folder path first — it's the most reliable signal:

| Folder path | Type |
|---|---|
| `src/components/*` | **Primitive** |
| `src/sections/*` | **Section** |
| anything with multiple section imports + full-page layout | **Composition page** |

If the folder path is ambiguous — read the `.tsx` imports to confirm:
- Imports only from `react`, own `.css`, `src/components/` → Primitive
- Imports from `src/sections/` or `src/components/` + folder is `src/sections/` → Section
- Imports multiple sections, root element is `<>` or `<main>` → Composition page

---

## Step 3: Check if sandbox entry exists

Before writing anything, check whether the target file already exists:

| Type | Target file |
|---|---|
| Primitive | `sandbox/main.tsx` |
| Section | `sandbox/sections.tsx` |
| Composition page | `sandbox/pages/[Name].tsx` + `sandbox/pages/[Name].html` |

**If the target file exists** → go to Step 4 (add to existing).
**If the target file does not exist** → go to Step 3a (create it first).

---

### Step 3a: Create missing sandbox entry

Only if the file does not exist. Read `tokens/tokens.css` to get the token prefix and detect dark mode support.

**If `sandbox/sandbox.css` doesn't exist — create it first.** Replace `[PREFIX]` with the actual token prefix.

#### `sandbox/sandbox.css`

```css
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html, body, #root {
  width: 100%;
  min-height: 100vh;
}

body {
  background: var(--[PREFIX]-color-background-default-default, #ffffff);
  color: var(--[PREFIX]-color-text-default-default, #1a1a1a);
  font-family: var(--[PREFIX]-typography-body-font-family, system-ui, sans-serif);
  font-size: var(--[PREFIX]-typography-body-size-medium, 16px);
  line-height: 1.5;
  transition: background 0.2s, color 0.2s;
}

/* Dark theme — target html so tokens cascade correctly into body */
html[data-theme="dark"] body {
  background: var(--[PREFIX]-color-background-default-default, #141414);
  color: var(--[PREFIX]-color-text-default-default, #f0f0f0);
}

/* --- Catalog pages (primitives + sections) --- */

.sandbox-page {
  width: 100%;
  min-height: 100vh;
  padding: 48px 40px;
}

.sandbox-header {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px 16px;
  width: 100%;
  margin-bottom: 48px;
  padding-bottom: 16px;
  border-bottom: 1px solid var(--[PREFIX]-color-border-default-default, rgba(0,0,0,0.1));
}

.sandbox-header h1 {
  font-size: 20px;
  font-weight: 700;
  flex-shrink: 0;
}

.sandbox-header-sub {
  font-size: 13px;
  opacity: 0.4;
  flex-shrink: 0;
}

.sandbox-header-right {
  margin-left: auto;
  display: flex;
  align-items: center;
  gap: 16px;
}

.sandbox-nav {
  display: flex;
  gap: 2px;
}

.sandbox-nav a {
  font-size: 13px;
  padding: 5px 12px;
  border-radius: 6px;
  color: inherit;
  text-decoration: none;
  opacity: 0.45;
  transition: opacity 0.15s, background 0.15s;
}

.sandbox-nav a:hover { opacity: 0.8; }

.sandbox-nav a.active {
  opacity: 1;
  font-weight: 600;
  background: rgba(0,0,0,0.06);
}

html[data-theme="dark"] .sandbox-nav a.active {
  background: rgba(255,255,255,0.08);
}

.sandbox-controls {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  opacity: 0.6;
}

.sandbox-controls select {
  padding: 3px 6px;
  border-radius: 5px;
  border: 1px solid var(--[PREFIX]-color-border-default-default, rgba(0,0,0,0.15));
  background: transparent;
  color: inherit;
  font-size: 13px;
  cursor: pointer;
}

.sandbox-section {
  width: 100%;
  margin-bottom: 56px;
}

.sandbox-section-title {
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  opacity: 0.35;
  margin-bottom: 20px;
  padding-bottom: 8px;
  border-bottom: 1px solid var(--[PREFIX]-color-border-default-default, rgba(0,0,0,0.08));
}

.sandbox-row {
  display: flex;
  flex-wrap: wrap;
  align-items: flex-start;
  gap: 12px;
  margin-bottom: 4px;
}

.sandbox-row-wrap {
  margin-bottom: 24px;
}

.sandbox-label {
  font-size: 11px;
  opacity: 0.35;
  margin-top: 6px;
  display: block;
}

/* --- Compositions index page --- */

.compositions-index {
  width: 100%;
  min-height: 100vh;
  padding: 48px 40px;
}

.compositions-index h1 {
  font-size: 24px;
  font-weight: 700;
  margin-bottom: 6px;
}

.compositions-index > p {
  font-size: 13px;
  opacity: 0.4;
  margin-bottom: 40px;
  padding-bottom: 20px;
  border-bottom: 1px solid var(--[PREFIX]-color-border-default-default, rgba(0,0,0,0.1));
}

.compositions-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
  gap: 12px;
}

.composition-card {
  display: block;
  padding: 20px 24px;
  border: 1px solid var(--[PREFIX]-color-border-default-default, rgba(0,0,0,0.1));
  border-radius: 10px;
  text-decoration: none;
  color: inherit;
  transition: border-color 0.15s, box-shadow 0.15s;
}

.composition-card:hover {
  border-color: var(--[PREFIX]-color-border-default-default, rgba(0,0,0,0.25));
  box-shadow: 0 2px 8px rgba(0,0,0,0.06);
}

.composition-card h3 {
  font-size: 15px;
  font-weight: 600;
  margin-bottom: 4px;
}

.composition-card p {
  font-size: 12px;
  opacity: 0.45;
}

@media (max-width: 768px) {
  .sandbox-page,
  .compositions-index { padding: 32px 20px; }
}

@media (max-width: 480px) {
  .sandbox-page,
  .compositions-index { padding: 24px 16px; }
  .sandbox-row { flex-direction: column; }
}
```

If dark mode NOT detected in tokens — omit both `html[data-theme="dark"]` blocks.

#### Creating `sandbox/main.tsx` (Primitives catalog)

```tsx
import React, { useState } from "react";
import ReactDOM from "react-dom/client";
import "../../tokens/tokens.css";
import "./sandbox.css";
import { [Name] } from "../[folder]/[Name]";

function Header() {
  const [theme, setTheme] = useState<"light" | "dark">("light");
  const handleTheme = (v: "light" | "dark") => {
    setTheme(v);
    document.documentElement.setAttribute("data-theme", v);
  };
  return (
    <div className="sandbox-header">
      <h1>Design System</h1>
      <span className="sandbox-header-sub">Primitives</span>
      <div className="sandbox-header-right">
        <nav className="sandbox-nav">
          <a href="./index.html" className="active">Primitives</a>
          <a href="./sections.html">Sections</a>
          <a href="./pages/index.html">Pages</a>
        </nav>
        <div className="sandbox-controls">
          <span>Theme</span>
          <select value={theme} onChange={(e) => handleTheme(e.target.value as "light" | "dark")}>
            <option value="light">Light</option>
            <option value="dark">Dark</option>
          </select>
        </div>
      </div>
    </div>
  );
}

export function Section({ title, children }: { title: string; children: React.ReactNode }) {
  return (
    <div className="sandbox-section">
      <div className="sandbox-section-title">{title}</div>
      {children}
    </div>
  );
}

export function Row({ label, children }: { label?: string; children: React.ReactNode }) {
  return (
    <div className="sandbox-row-wrap">
      <div className="sandbox-row">{children}</div>
      {label && <span className="sandbox-label">{label}</span>}
    </div>
  );
}

function App() {
  return (
    <div className="sandbox-page">
      <Header />
      [FIRST_SECTION_BLOCK]
    </div>
  );
}

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode><App /></React.StrictMode>
);
```

If dark mode NOT detected in tokens — remove the theme `<select>` and simplify Header.

Also create `sandbox/index.html` if missing:
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Design System — Primitives</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="./main.tsx"></script>
  </body>
</html>
```

---

#### Creating `sandbox/sections.tsx` (Sections catalog)

Same structure as `main.tsx` but:
- Title: `"Primitive components"` → `"Sections"`
- Active nav link: `index.html` → `sections.html`
- Import from `src/sections/` instead of `src/components/`

Also create `sandbox/sections.html` if missing (same as `index.html` but `src="./sections.tsx"`).

---

#### Creating `sandbox/pages/[Name].tsx` + `sandbox/pages/[Name].html` (Composition page)

Check that all sections this page imports exist in `src/sections/`. If any are missing — stop:

> To add this page I need these sections which don't exist yet:
> - [SectionName]
>
> Run `add-to-ds` with a Figma link for each missing section first, then run `add-to-sandbox` again.

Create `sandbox/pages/[Name].html`:
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Design System — [Name]</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="./[Name].tsx"></script>
  </body>
</html>
```

Create `sandbox/pages/[Name].tsx` — copy source, adjust import paths relative to `sandbox/pages/`:
```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import "../../../tokens/tokens.css";
import "../../sandbox.css";
import "../../../src/layout.css";
import { SectionA } from "../../../src/sections/SectionA/SectionA";
// ...other sections

function App() {
  return (
    <>
      <SectionA />
    </>
  );
}

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode><App /></React.StrictMode>
);
```

Also create `sandbox/pages/index.html` + `sandbox/pages/index.tsx` if missing — use the same template as the pages index in Step 3a. Register this page as the first entry in the `pages` array.

---

## Step 4: Add to existing sandbox entry

The target file exists. Use `str_replace` only — never rewrite the file.

---

### Route A — Adding primitive to `sandbox/main.tsx`

Read the file in full. Make two surgical edits:

**Edit 1 — Add import** after the last existing import:
```tsx
import { [Name] } from "../[folder]/[Name]";
```

**Edit 2 — Add Section block** before the closing `</div>` in `App()`:
```tsx
<Section title="[Name]">
  <Row label="[Variant] — default / hover / disabled">
    <[Name] prop="value" />
    <[Name] prop="value" state="hover" />
    <[Name] prop="value" state="disabled" />
  </Row>
  {/* one Row per variant — infer from props in the .tsx file */}
</Section>
```

**Layout rules:**
- One `<Row>` = one variant, all states side by side
- Separate rows for different variants or sizes
- Label on the row, not on individual items
- Infer variant matrix from props defined in the source `.tsx`

---

### Route B — Adding section to `sandbox/sections.tsx`

Read the file in full. Make two surgical edits:

**Edit 1 — Add import:**
```tsx
import { [Name] } from "../src/sections/[Name]/[Name]";
```

**Edit 2 — Add Section block** inside `App()`:
```tsx
<Section title="[Name]">
  <div style={{ width: "100%" }}>
    <[Name] />
  </div>
</Section>
```

If the section accepts meaningful variants (e.g. different backgrounds, with/without nav) — show each as a separate `<Row>` with a label. Infer from props in the source `.tsx`.

---

### Route C — Adding composition page to `sandbox/pages/`

Already handled in Step 3a — page files are always new files, never modified.

After creating the page files, register it in `sandbox/pages/index.tsx` via `str_replace`:

```tsx
// add to pages array:
{ name: "[Name]", description: "[one sentence: what sections this page contains]", href: "./[Name].html" },
```

---

## Step 5: Verify

- Only the correct target file(s) were created or modified
- All previously existing content is still intact
- No files in `src/`, `tokens/`, or unrelated sandbox files were touched
- All imports resolve to existing files

Report:

**Route A:**
> Added `[Name]` to `sandbox/main.tsx`.
> http://localhost:5173/sandbox/index.html

**Route B:**
> Added `[Name]` to `sandbox/sections.tsx`.
> http://localhost:5173/sandbox/sections.html

**Route C:**
> Page created: `sandbox/pages/[Name].html`
> Registered in pages index.
> http://localhost:5173/sandbox/pages/[Name].html

---

## Step 6: Open in browser

Determine the URL based on route:

| Route | URL |
|---|---|
| A — Primitive | `http://localhost:5173/sandbox/index.html` |
| B — Section | `http://localhost:5173/sandbox/sections.html` |
| C — Page | `http://localhost:5173/sandbox/pages/[Name].html` |

**Check if the dev server is already running:**

Run `lsof -ti:5173` (or equivalent).

- **Server is running** — open the URL in Cursor Simple Browser directly. Do NOT run `npm run sandbox:dev`.
- **Server is not running** — run `npm run sandbox:dev`, wait for it to start, then open the URL.

In both cases, report only the direct link:

> Done. View at: [URL]