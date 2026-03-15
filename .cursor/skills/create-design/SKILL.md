---
name: create-design
description: Vibe-code one or more pages on a given theme using only existing design system components. Use when the user describes a page or set of pages in natural language without providing a Figma link. Strictly uses only what exists in src/components/ and src/sections/ — never invents UI. If something is missing, offers to add it via add-to-ds or warns about improvisation risk.
---

# Create Design

Assembles one or more pages from natural language description using only what already exists in the design system.

**Core rule:** every visual element must come from `src/components/` or `src/sections/`. The agent never invents new UI, never writes inline styles to fake a component, never approximates something that doesn't exist. If a required piece is missing — it stops and asks.

---

## Required Input

A description of what to build. Examples:

```
Create a landing page for a SaaS product
Build an e-commerce catalog page and a product detail page
Make a login page and a signup page
```

No Figma link needed — this is vibe-coding from description. If the user also provides a Figma link, use `add-to-ds` instead.

---

## Step 1: Understand what's needed

Read the description and decompose it into pages and sections.

For each page, list the sections it logically needs:

**Example — "SaaS landing page":**
```
HomePage
├── Header          (navigation, logo, auth buttons)
├── HeroBasic       (headline, subline, CTA)
├── FeatureGrid     (3-column feature cards)
├── PricingSection  (pricing cards)
└── Footer          (links, copyright)
```

Use common layout patterns as a guide:
- Every page needs a Header and Footer
- Landing pages: Hero + features/benefits + CTA + Footer
- Catalog pages: Header + filter bar + product grid + pagination + Footer
- Detail pages: Header + product info + related items + Footer
- Auth pages: Header (minimal) + centered form + Footer (minimal)
- Dashboard: Sidebar nav + content area + Header

List all sections needed for all requested pages before moving to Step 2.

---

## Step 2: Audit the design system

Scan `src/components/` and `src/sections/` against the list from Step 1.

Build a full inventory:

```
COMPONENTS (src/components/)
✅ Button
✅ Input
✅ Badge
❌ PricingCard    — missing
❌ FeatureCard    — missing

SECTIONS (src/sections/)
✅ Header
✅ Footer
❌ HeroBasic      — missing
❌ FeatureGrid    — missing
❌ PricingSection — missing
```

---

## Step 3: Handle missing pieces

Present the inventory to the user with clear options.

### If only a few items are missing:

> To build the pages you described I have everything except:
>
> **Missing sections:**
> - HeroBasic — main hero block with headline and CTA
> - FeatureGrid — 3-column grid of feature cards
> - PricingSection — pricing tier cards
>
> **Missing components:**
> - PricingCard — individual pricing tier card
>
> How should I handle the missing pieces?
>
> **Option 1 — Add from your design system (recommended)**
> Share Figma links to the missing items and I'll build them properly via `add-to-ds` before assembling the pages. Result will match your DS exactly.
>
> **Option 2 — I'll improvise ⚠️**
> I'll create the missing sections using only existing tokens and components as building blocks. The layout and structure will be my own interpretation — not based on any Figma design. Visual divergence from your DS may be significant. Proceed at your own risk.
>
> Which option?

### If most things are missing (more than half the needed sections):

> The pages you described require many sections that don't exist in the DS yet:
> [list]
>
> I recommend building the DS first before assembling pages.
> Share Figma links and I'll add everything via `add-to-ds`.
>
> Or I can build all pages with improvised sections, but the result will diverge significantly from your design system. ⚠️

### If everything exists:

Proceed directly to Step 4 — no questions needed.

---

## Step 4: Build missing pieces (if Option 1 chosen)

For each Figma link provided — run `add-to-ds` in full.

This includes:
- Component/section type detection
- Dependency checks
- Icon handling
- Token validation
- Responsive assessment
- Tailwind → CSS Modules conversion

Only proceed to Step 5 after all missing pieces are built and validated.

---

## Step 5: Improvise missing pieces (if Option 2 chosen)

For each missing section — build it using only:
- Existing components from `src/components/`
- Tokens from `tokens/tokens.css`
- Standard CSS layout (flexbox, grid)

**Rules for improvised sections:**
- No hardcoded colors — only `var(--[prefix]-*)` tokens
- No raw pixel values for spacing — only size tokens
- Use only existing components — no new primitives
- Keep layout generic and simple — no decorative flourishes
- Add a comment at the top of each improvised file:

```tsx
// ⚠️ IMPROVISED SECTION — not based on a Figma design.
// This was created by the AI agent without a design reference.
// Review and align with your design system before production use.
```

---

## Step 6: Assemble pages

For each page — create `sandbox/pages/[PageName].tsx` + `sandbox/pages/[PageName].html`.

**Page file structure:**
```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import "../../../tokens/tokens.css";
import "../../sandbox.css";
// import "../../../src/layout.css"; — only if file exists
import { Header } from "../../../src/sections/Header/Header";
import { HeroBasic } from "../../../src/sections/HeroBasic/HeroBasic";
import { Footer } from "../../../src/sections/Footer/Footer";
// ...other sections in page order

function App() {
  return (
    <>
      <Header />
      <HeroBasic
        headline="Your headline here"
        subline="Supporting copy that explains the value"
        ctaLabel="Get started"
      />
      <Footer />
    </>
  );
}

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode><App /></React.StrictMode>
);
```

**Content rules:**
- Pass realistic placeholder content as props — not "Lorem ipsum"
- Match content to the page theme: SaaS → tech product copy, e-commerce → product names, etc.
- Never hardcode content inside section components — always pass as props

**HTML entry file:**
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
    <script type="module" src="./[PageName].tsx"></script>
  </body>
</html>
```

---

## Step 7: Register all pages in the index

Read `sandbox/pages/index.tsx`. Use `str_replace` to add all new pages to the `pages` array in one edit:

```tsx
{ name: "Home", description: "Hero + features + pricing + footer", href: "./HomePage.html" },
{ name: "Catalog", description: "Product grid with filters and pagination", href: "./CatalogPage.html" },
```

If `sandbox/pages/index.tsx` doesn't exist — create it and `sandbox/pages/index.html` using the template from `add-to-sandbox`.

---

## Step 8: Open in browser

Check if port 5173 is in use (`lsof -ti:5173`):
- **Running** → open the first created page in Cursor Simple Browser. Do NOT run `npm run sandbox:dev`.
- **Not running** → run `npm run sandbox:dev`, wait for start, then open.

Report all created pages:

> Pages ready:
> - Home: http://localhost:5173/sandbox/pages/HomePage.html
> - Catalog: http://localhost:5173/sandbox/pages/CatalogPage.html
>
> Pages index: http://localhost:5173/sandbox/pages/index.html
>
> ⚠️ Improvised sections (review before production):
> - HeroBasic, FeatureGrid — not based on Figma designs