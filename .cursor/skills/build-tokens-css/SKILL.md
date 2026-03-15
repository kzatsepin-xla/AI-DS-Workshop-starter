---
name: build-tokens-css
description: Scan the tokens/ folder for JSON files exported from Figma, resolve all alias references, and produce a single tokens/tokens.css file. Also works via Figma MCP if a URL is provided instead of JSON files. Use when the user asks to sync tokens, build tokens.css, or drops JSON files into the tokens/ folder.
---

# Build tokens.css

Scans `tokens/` for JSON files, resolves aliases, writes one `tokens/tokens.css`. Works with any export format from any Figma plugin that produces W3C-style or flat JSON token files.

Also supports Figma MCP if a URL is provided — see Mode B below.

**If `tokens/tokens.css` already exists:** overwrites it completely with the fresh result.

## Required Inputs

Check what the user provided:

**If JSON files are present in `tokens/`** — proceed immediately with Mode A. No questions needed.

**If a Figma URL was provided** — use Mode B (MCP).

**If neither** — ask:

> Drop your exported token JSON files into the `tokens/` folder and let me know, or share a Figma file URL to pull tokens via MCP.

---

## Mode A — JSON files in tokens/

### Step 1: Scan tokens/ for JSON files

List all `*.json` files in `tokens/`. Read every file.

Classify each file by its role:
- **Primitives** — file name contains `primitive`, `base`, `global`, `raw`, or values are plain hex/numbers with no `{...}` references. These define the raw palette.
- **Semantic light** — file name contains `light`, `day`, or is the default semantic file. Values reference primitives via `{Group.Scale}` aliases.
- **Semantic dark** — file name contains `dark`, `night`. Same structure as light but different values.
- **Size / spacing** — file name contains `size`, `space`, `spacing`, `radius`, `dimension`.
- **Typography** — file name contains `typography`, `type`, `font`.
- **Effects** — file name contains `effect`, `shadow`, `blur`.

If classification is ambiguous — infer from the structure: files with all plain values = primitives, files with `{...}` alias values = semantic.

### Step 2: Build the primitive lookup table

From all primitive files, build a flat map:
```
"Group.Scale" → "#hexvalue"
...
```

Key format: `"GroupName.Scale"` matching the `{Group.Scale}` alias syntax used in semantic files.

### Step 3: Resolve all aliases

For every token in semantic, size, and typography files:
- If value is `"{Group.Scale}"` — look it up in the primitive table and replace with the resolved hex value
- If value is already a plain hex/number — keep as-is
- If alias cannot be resolved — keep the raw alias string and log a warning at the end

### Step 4: Convert JSON keys to CSS variable names

Token names are derived from the JSON path + file prefix. Rules:

1. Start with the file's top-level key or file name prefix — strip it if it identifies a theme (light/dark), keep it if it identifies a category (size, typography, color)
2. Join path segments with `-` and lowercase everything: `Background > Default > Default` → `background-default-default`
3. Replace spaces with `-`
4. Prepend the project prefix detected from existing `tokens.css`. If no prefix found, derive it from the file name or JSON top-level key.

Spaces in key names — replace spaces with `-` and lowercase.

### Step 5: Detect dark mode

If a semantic dark file exists — collect all tokens where the resolved value differs from the light equivalent. Only differing tokens go into `[data-theme="dark"]`.

### Step 6: Write tokens.css

```css
/* ==========================================================================
   Design Tokens
   Generated from Figma token export. Do not edit manually.
   Last synced: [date]
   ========================================================================== */

/* --- Size tokens (theme-independent) --- */
:root {
  --prefix-size-space-100: 4px;
  --prefix-size-space-200: 8px;
  /* ... */
}

/* --- Typography tokens (theme-independent) --- */
:root {
  --prefix-typography-body-font-family: YourFont, sans-serif;
  /* ... */
}

/* --- Color tokens — light theme (default) --- */
:root,
[data-theme="light"] {
  --prefix-color-background-brand-default: #value;
  /* ... */
}

/* --- Color tokens — dark theme --- */
[data-theme="dark"] {
  --prefix-color-background-brand-default: #value;
  /* ... */
}
```

Value formatting:
- Numeric values without unit (e.g. `"8"`) → append `px`, unless token name suggests unitless (`font-weight`, `opacity`, `z-index`, `line-height`)
- Font family → append `, sans-serif` if not already present
- Colors → write as-is (hex, rgba)
- If dark block is empty → include it with a comment: `/* No dark mode tokens found in exported files */`

### Step 7: Audit

Run:
```
npm run token-usage:audit
```

Report:
- [ ] `tokens/tokens.css` written successfully
- [ ] Total token count (size + typography + color light + color dark)
- [ ] Any unresolved aliases (warnings)
- [ ] Any missing tokens reported by the audit

---

## Mode B — Figma MCP

Use this mode when the user provides a Figma file URL instead of JSON files.

### Step 1: Connect to Figma MCP

Try `get_metadata` with the fileKey. If it fails — ask:

> Figma MCP is not connected. Please provide the MCP server URL (usually `http://127.0.0.1:3845/mcp` for the Figma desktop app).

### Step 2: Discover nodes

Extract `fileKey` from the URL. Call `get_metadata` on the file to find pages. Look for pages related to components, foundations, or tokens — names vary by project.

Collect node IDs from as many different component types as possible — aim for at least 10–15 different node types. Separately collect node IDs from frames whose names suggest dark mode.

### Step 3: Pull variables — light and dark separately

Call `get_variable_defs(fileKey, nodeId)` on all collected nodes.

- Light/default nodes → `lightTokens` map
- Dark frame nodes → `darkTokens` map

Keep calling until two consecutive calls per theme produce no new token names.

Dark overrides = tokens present in both maps with different values.

### Step 4: Assess token quality

Before writing, check:
- No consistent prefix → warn
- Purely presentational names (raw color/value names with no semantic meaning) → warn
- Fewer than 5 tokens total → warn
- Missing size or typography groups → warn

If critical issues found — report to user and ask whether to proceed or fix in Figma first.

### Step 5: Write tokens.css and audit

Same as Mode A Steps 6–7, but no JSON files involved.

---

## When to re-run

- JSON files dropped into `tokens/` after a new Figma export
- Figma Variables updated and need to be re-synced via MCP
- `tokens/tokens.css` is missing or corrupted
