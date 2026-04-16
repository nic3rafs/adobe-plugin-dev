---
name: adobe-plugin-dev
description: >
  Build Adobe Creative Cloud plugins with UXP (Unified Extensibility Platform) from scratch to production.
  Use this skill whenever the user wants to create, scaffold, build, debug, or publish a plugin for any Adobe
  Creative Cloud app — InDesign, Photoshop, Illustrator, Premiere Pro, XD, After Effects, or any other UXP-supported
  app. Triggers on: "build an InDesign plugin", "create an Adobe plugin", "UXP plugin", "add a panel to InDesign",
  "automate InDesign with a plugin", "make a Photoshop extension", "Adobe extension development", "UXP script",
  "migrate from CEP", or any mention of building something that runs inside an Adobe app. Even if the user says
  "script" or "extension" instead of "plugin", use this skill — those are often the same thing in this context.
---

# Adobe Plugin Development (UXP)

You are helping build a production-quality Adobe UXP plugin. UXP (Unified Extensibility Platform) is Adobe's modern JavaScript-based framework for extending Creative Cloud apps. It replaces the old CEP/ExtendScript approach with a faster, more secure, and more capable system.

## How to use this skill

1. **Identify the target app** — Ask which Adobe app the plugin is for if not stated. Start with the app-specific reference file.
2. **Identify the plugin type** — Command (modal) vs. Panel (persistent sidebar) vs. Script (.idjs file). Ask if unclear.
3. **Read the right reference file** before writing any code:
   - InDesign → read `references/indesign.md`
   - Photoshop → read `references/photoshop.md` (add when needed)
   - Illustrator → read `references/illustrator.md` (add when needed)
   - Shared UXP patterns → read `references/uxp-common.md`
4. **Follow the workflow below** to scaffold, build, and ship.

---

## Currently supported apps

| App | Reference file | Status |
|-----|---------------|--------|
| InDesign | `references/indesign.md` | ✅ Full coverage |
| Photoshop | `references/photoshop.md` | 🔜 Add when needed |
| Illustrator | `references/illustrator.md` | 🔜 Add when needed |
| Premiere Pro | `references/premiere-pro.md` | 🔜 Add when needed |
| After Effects | `references/after-effects.md` | 🔜 Add when needed |

When the user wants to add a new app, read `references/uxp-common.md` and use it as the foundation — then research that app's specific DOM/API layer.

---

## Universal UXP Architecture

All UXP plugins share this foundation regardless of the target app.

### Plugin folder structure

```
my-plugin/
├── manifest.json          ← Required. Describes the plugin to UXP.
├── index.html             ← Main UI entry (for panel/HTML-based plugins)
├── index.js               ← Main JS entry (for command-only or JS-entry plugins)
├── icons/
│   ├── dark@1x.png        ← 23×23 icon for dark themes
│   ├── dark@2x.png        ← 46×46 icon for dark themes (retina)
│   ├── light@1x.png       ← 23×23 icon for light themes
│   └── light@2x.png       ← 46×46 icon for light themes (retina)
└── package.json           ← Only needed if using a build step (React, SWC, etc.)
```

For React/SWC builds, add a `src/` folder and `webpack.config.js` — the built output goes into `dist/` which is what UXP loads.

### manifest.json — the core contract

Every plugin needs exactly one `manifest.json` at the root. This file tells UXP everything about your plugin.

```json
{
  "id": "com.yourcompany.pluginname",
  "name": "My Plugin",
  "version": "1.0.0",
  "manifestVersion": 5,
  "main": "index.html",
  "host": {
    "app": "ID",
    "minVersion": "18.5.0"
  },
  "entrypoints": [
    {
      "type": "panel",
      "id": "mainPanel",
      "label": { "default": "My Panel" },
      "minimumSize": { "width": 230, "height": 200 },
      "maximumSize": { "width": 2000, "height": 2000 },
      "preferredDockedSize": { "width": 300, "height": 300 },
      "preferredFloatingSize": { "width": 300, "height": 300 },
      "icons": [
        { "width": 23, "height": 23, "path": "icons/dark@1x.png",
          "scale": [1, 2], "theme": ["darkest", "dark", "medium"] },
        { "width": 23, "height": 23, "path": "icons/light@1x.png",
          "scale": [1, 2], "theme": ["lightest", "light"] }
      ]
    }
  ],
  "icons": [
    { "width": 48, "height": 48, "path": "icons/plugin@1x.png",
      "scale": [1, 2], "theme": ["all"], "species": ["pluginList"] }
  ],
  "requiredPermissions": {
    "network": {
      "domains": ["https://api.example.com"]
    },
    "localFileSystem": "request",
    "clipboard": "readAndWrite"
  }
}
```

**Key manifest fields:**
- `id` — reverse-domain style, must be unique. Get a real one from the Adobe Developer Distribution portal before publishing.
- `manifestVersion` — use `5` (current as of InDesign 18.5+)
- `main` — `"index.html"` for panel plugins, or `"index.js"` for command-only plugins
- `host.app` — app code: `"ID"` = InDesign, `"PS"` = Photoshop, `"AI"` = Illustrator, `"PPRO"` = Premiere Pro
- `host.minVersion` — the minimum host app version required
- `entrypoints` — `"command"` (appears in menus, creates modal dialogs) or `"panel"` (persistent side panel)

**Multi-host** (same plugin for multiple apps):
```json
"host": [
  { "app": "ID", "minVersion": "18.5.0" },
  { "app": "PS", "minVersion": "24.1.0" }
]
```

**Permissions reference:**
```json
"requiredPermissions": {
  "network": {
    "domains": ["https://api.example.com", "https://other.com"]
    // or "domains": "all"  (use sparingly — future users will see this)
  },
  "localFileSystem": "plugin",   // sandbox only (safest)
  // "localFileSystem": "request", // open/save via file picker
  // "localFileSystem": "fullAccess", // unrestricted (requires justification)
  "clipboard": "readAndWrite",   // or "read"
  "ipc": { "enablePluginCommunication": true },  // inter-plugin messaging
  "webview": {
    "allow": "yes",
    "domains": ["https://my-embedded-app.com"],
    "enableMessageBridge": "localAndRemote"
  },
  "allowCodeGenerationFromStrings": false
}
```

**Important:** Changes to manifest.json always require a full unload + reload in UDT (not just a watch-reload).

---

## Entrypoint wiring (index.js)

This is how you connect manifest entrypoints to your JavaScript:

```javascript
const { entrypoints } = require("uxp");

entrypoints.setup({
  // Optional lifecycle hooks for the whole plugin
  plugin: {
    create(plugin) { /* plugin loaded */ },
    destroy()      { /* plugin unloaded */ }
  },

  // Command entrypoints (menu items → modal dialogs)
  commands: {
    myCommandId: () => runMyCommand()
  },

  // Panel entrypoints (persistent side panels)
  panels: {
    myPanelId: {
      create()           { /* panel DOM created */ },
      show({ node } = {}) { /* panel became visible; node = container element */ },
      hide()             { /* panel hidden */ },
      destroy()          { /* panel removed */ },
      invokeMenu(id)     { /* panel flyout menu item clicked */ }
    }
  }
});
```

The IDs in `entrypoints.setup()` must exactly match the `"id"` fields in your manifest entrypoints.

---

## UI approaches

UXP supports three ways to build UI. You can mix them.

**1. Plain HTML + Spectrum UXP widgets (simplest)**
```html
<!DOCTYPE html>
<html>
<head><script src="index.js"></script></head>
<body>
  <sp-heading>My Plugin</sp-heading>
  <sp-body>Ready to go.</sp-body>
  <sp-button id="runBtn" variant="cta">Run</sp-button>
</body>
</html>
```
Built-in Spectrum UXP tags: `sp-button`, `sp-textfield`, `sp-checkbox`, `sp-radio`, `sp-dropdown`, `sp-slider`, `sp-heading`, `sp-body`, `sp-label`, `sp-divider`, `sp-link`, `sp-progressbar`, `sp-icon`.

**2. Spectrum Web Components (modern, more flexible)**
Requires `"featureFlags": { "enableSWCSupport": true }` in manifest.
```bash
yarn add @spectrum-web-components/theme @swc-uxp-wrappers/button
```
```javascript
import "@spectrum-web-components/theme/sp-theme.js";
import "@swc-uxp-wrappers/button/sp-button.js";
```
```html
<sp-theme theme="spectrum" color="light" scale="medium" dir="ltr">
  <sp-button variant="primary">Click me</sp-button>
</sp-theme>
```

**3. React (most powerful for complex UIs)**
See `references/uxp-common.md` → React section for the full setup. Key pattern: webpack bundles your JSX, `uxp` and `indesign` are marked as externals so they're loaded from the host app at runtime.

**CSS limitations in UXP** — UXP's CSS engine is NOT a full browser. Things that don't work:
- `display: grid` — not supported
- `gap` with flexbox — not supported
- `float` — not supported
- `aspect-ratio` — not supported
- CSS animations (`transition`, `@keyframes`) — unreliable, avoid
- Use `flexbox` + `position` for layouts

---

## Development workflow

### Step 1: Install tools
- **InDesign 2023 (v18.5+)** or the target app installed
- **UXP Developer Tools (UDT)** — install from Creative Cloud Desktop → All Apps → UXP Developer Tools
- **Node.js + npm/yarn** — required for React/SWC builds
- **VS Code** (recommended) with the UXP extension

### Step 2: Create the plugin
Option A — scaffold manually (fastest for simple plugins): create `manifest.json` + `index.html`/`index.js`.

Option B — clone a starter from `https://github.com/AdobeDocs/uxp-indesign-samples`:
- `plugins/command-plugin-starter` — command-only, no HTML
- `plugins/panel-plugin-starter` — panel with HTML UI
- `plugins/react-starter` — panel with React + webpack
- `plugins/swc-uxp-starter` — panel with Spectrum Web Components
- `plugins/webview-sample` — panel using an embedded WebView

### Step 3: Load in UDT
1. Open UDT
2. Click **Add Plugin** → select your plugin folder (the folder containing manifest.json)
3. Click **Load** to load it into the host app
4. Use **Watch** to auto-reload on file changes (file changes only — manifest changes need full reload)
5. Use **Debug** to open Chrome DevTools-like debugger (console, breakpoints, element inspector)

### Step 4: Build loop
- Edit your JS/HTML/CSS
- UDT Watch auto-reloads → test in the app
- Use `console.log()` — visible in UDT debugger console
- Fix errors, repeat

### Step 5: Package and distribute
```
UDT → Actions → Package
```
This creates a `.ccx` file (it's a zip). Before packaging for real distribution:
1. Get a plugin ID from [Adobe Developer Distribution Portal](https://developer.adobe.com/developer-distribution/)
2. Put that ID in `manifest.json` → `"id"` field
3. Package via UDT

**Distribution options:**
- **Private / direct** — share the `.ccx` file; users double-click to install via Creative Cloud
- **Enterprise** — deploy via managed configuration in Creative Cloud for Teams/Enterprise
- **Creative Cloud Marketplace** — submit via the Developer Distribution Portal for public listing (review process required)

---

## App-specific references

Read the relevant reference file for the target app's DOM API and unique patterns:

- **InDesign**: `references/indesign.md` — DOM/OMV API, documents, pages, text frames, tables, styles, events
- **Photoshop** (future): `references/photoshop.md`
- **Illustrator** (future): `references/illustrator.md`

For shared UXP patterns (file system, network, storage, React setup): `references/uxp-common.md`

---

## Common pitfalls

- **Manifest changes** need a full unload/reload, not just Watch-reload
- **`require("indesign")`** is only available in InDesign — conditionally guard multi-host plugins
- **Collections use `.item(N)`** not `[N]` — `story.paragraphs.item(0)` not `story.paragraphs[0]`
- **`===` doesn't work for InDesign DOM objects** — use `.equals()` instead
- **`instanceof` doesn't work** for InDesign DOM objects — use `.constructor.name` or `constructorName`
- **`display: grid`** not supported in UXP CSS — use flexbox
- **`DOMParser`, `TextDecoder`, `structuredClone`** are not available in UXP
- **The global `document`** in InDesign scripts refers to the HTML document (UI), not the InDesign document — use `require("indesign").app.activeDocument` for InDesign documents
- **Promises that never resolve** often mean a missing permission in manifest — add the needed `requiredPermissions` entry
