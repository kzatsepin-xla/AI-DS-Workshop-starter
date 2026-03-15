---
name: init-ds-project
description: Bootstrap a design system project from scratch. Use when the user wants to start a new DS project, scaffold a component library, or says "start from scratch", "new project", "init DS". Creates project structure, scripts, and package.json only. Does NOT create sandbox files or tokens — those are handled by add-to-sandbox and build-tokens-css.
---

# Initialize Design System Project

Scaffolds the project structure, scripts, and configuration. Nothing else.

**Does NOT create:**
- `tokens/tokens.css` or any files inside `tokens/` — use `build-tokens-css`
- `sandbox/` files — use `add-to-sandbox`

## Required Inputs

No mandatory inputs. If a project name was specified — use it. Otherwise use `my-design-system`.

Proceed immediately without asking.

## Default Stack

- **Framework**: React 19 + ReactDOM
- **Language**: TypeScript (Vite handles config)
- **Styling**: Plain `.css` files colocated with components
- **Bundler**: Vite (latest)
- **Package manager**: npm
- **Token source**: JSON files or Figma MCP — handled separately by `build-tokens-css`
- **Theme switching**: `[data-theme="light"]` / `[data-theme="dark"]` on `:root`
- **No Code Connect**: skip entirely

## Workflow

```
- [ ] Step 1: Scaffold project structure
- [ ] Step 2: Set up validation scripts
- [ ] Step 3: Install dependencies
- [ ] Step 4: Validate
```

### Step 1: Scaffold Project Structure

Create the following — nothing inside `tokens/` or `sandbox/`:

```
project-root/
├── src/
│   └── components/           # ComponentName/ComponentName.tsx + ComponentName.css
├── tokens/                   # Empty — populated by build-tokens-css
├── sandbox/                  # Empty — populated by add-to-sandbox
├── scripts/
│   ├── check-token-usage.mjs
│   └── dev.mjs
├── .gitignore
├── package.json
└── README.md
```

**`.gitignore`:**
```
node_modules/
dist/
```

**`package.json`:**

```json
{
  "private": true,
  "type": "module",
  "scripts": {
    "token-usage:audit": "node scripts/check-token-usage.mjs",
    "token-usage:check": "node scripts/check-token-usage.mjs --strict",
    "ds:check": "npm run token-usage:audit",
    "ds:check:full": "npm run token-usage:audit",
    "sandbox:dev": "vite --config vite.config.js --root sandbox",
    "dev": "node scripts/dev.mjs"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "typescript": "^5.9.0",
    "vite": "^7.1.0",
    "vite-plugin-svgr": "^4.0.0"
  }
}
```

**`README.md`** — include the project name and this order of next steps:
1. `build-tokens-css` — populate `tokens/tokens.css`
2. `add-to-sandbox` — set up the sandbox dev server
3. `add-to-ds` — create the first component

**`vite.config.js`:**

```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import svgr from "vite-plugin-svgr";

export default defineConfig({
  plugins: [react(), svgr()],
});
```

**Critical:** `sandbox:dev` runs `vite --config vite.config.js --root sandbox` — the `--config` flag points to the root-level config explicitly. Without it, Vite looks for config inside `sandbox/` and misses the svgr plugin, causing `?react` SVG imports to return data URI strings instead of React components → `InvalidCharacterError` at runtime.

### Step 2: Set Up Validation Scripts

**`scripts/check-token-usage.mjs`:**

1. Reads `tokens/tokens.css` — if file doesn't exist, exits with a helpful message asking to run `build-tokens-css` first.
2. Extracts all defined `--*` custom property names from `tokens.css`.
3. Reads all `*.css` files in `src/components/`.
4. Extracts every `var(--*)` reference used in components.
5. Reports tokens referenced in components but not defined in `tokens/tokens.css`.
6. `--strict` flag: exit 1 on any missing token.

The script must detect the token prefix dynamically — do not hardcode any prefix.

**`scripts/dev.mjs`:**

Runs `sandbox:dev` (Vite). Simple wrapper:

```js
import { spawn } from "child_process";
const proc = spawn("npm", ["run", "sandbox:dev"], { stdio: "inherit", shell: true });
proc.on("exit", (code) => process.exit(code));
```

### Step 3: Install Dependencies

Run `npm install`.

### Step 4: Validate

- [ ] All folders exist: `src/components/`, `tokens/` (empty), `sandbox/` (empty), `scripts/`
- [ ] `package.json` is valid
- [ ] `npm install` completes without errors
- [ ] `scripts/check-token-usage.mjs` exists

**Next steps — tell the user explicitly:**

> Project scaffolded. Run these next:
> 1. **`build-tokens-css`** — drop Figma JSON exports into `tokens/` and run, or provide a Figma URL
> 2. **`add-to-sandbox`** — creates sandbox entry points automatically on first use
> 3. **`add-to-ds`** — create your first component (give it a Figma link)