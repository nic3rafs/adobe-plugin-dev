# InDesign UXP Plugin Reference

Complete reference for building UXP plugins and scripts for Adobe InDesign.

## Table of Contents
1. [Requirements & versions](#requirements)
2. [Plugin types: command vs panel vs script](#plugin-types)
3. [Accessing the InDesign DOM](#accessing-the-indesign-dom)
4. [Document model: key objects & hierarchy](#document-model)
5. [Working with text](#working-with-text)
6. [Working with graphics and page items](#graphics)
7. [Styles](#styles)
8. [Events and listeners](#events)
9. [Menus](#menus)
10. [Dialogs (DOM-based)](#dialogs)
11. [File operations](#file-operations)
12. [Network requests](#network)
13. [Storage (plugin data persistence)](#storage)
14. [Scripts (.idjs)](#scripts)
15. [React plugin setup](#react-setup)
16. [Known limitations and gotchas](#gotchas)
17. [ExtendScript → UXP migration cheatsheet](#migration)

---

## 1. Requirements {#requirements}

- **InDesign 2023 (v18.0+)** — UXP scripts first appeared
- **InDesign 2023.1 (v18.4+)** — InDesign DOM moved to module (`require("indesign")`)
- **InDesign 2023.2 (v18.5+)** — Stable baseline for plugins; use `"minVersion": "18.5.0"` in manifest
- **UXP Developer Tools (UDT) v1.9+** — install from Creative Cloud Desktop
- **Node.js 14+** — only if using build tools (React, SWC)
- `"manifestVersion": 5` — current version; required for all new plugins

---

## 2. Plugin types: command vs panel vs script {#plugin-types}

### Command plugin
- Appears as a menu item in InDesign (under Plugins menu)
- When triggered, runs JavaScript and can open a modal dialog
- Entry: `"type": "command"` in manifest
- No persistent UI — everything happens inside the command handler
- Main entry can be `index.js` (no HTML needed)

### Panel plugin
- Persistent sidebar panel like any InDesign panel (Layers, Pages, etc.)
- Always visible when open; survives across documents
- Entry: `"type": "panel"` in manifest
- Requires HTML UI (`"main": "index.html"`)
- Panel lifecycle: `create → show → hide → show → ... → destroy`

### Script (.idjs)
- A single JavaScript file dropped into the Scripts panel or run via InDesign
- No manifest, no install needed — just run it
- Can show a modal dialog using `app.dialogs.add()` only
- Supports `await` at top level (global await)
- Cannot create persistent panels
- Good for automation tasks, data import/export, batch processing
- Extension: `.idjs`

**When to use which:**
- Automating a one-off task → Script
- Batch processing with no UI → Script
- Tool the user will use regularly with a side panel → Panel plugin
- Adding a menu item for a specific workflow action → Command plugin
- Complex UI with persistent state → Panel plugin (possibly with React)

---

## 3. Accessing the InDesign DOM {#accessing-the-indesign-dom}

Since InDesign v18.4, the DOM is a module. Always use `require()`:

```javascript
const indesign = require("indesign");
const app = indesign.app;

// Or destructure:
const { app, ScriptLanguage, UserInteractionLevels } = require("indesign");
```

**Never rely on globals like `app` or `Document` being available** — they were in old ExtendScript but not reliably in UXP.

### DOM versioning (forward compatibility)

You can pin a specific DOM version to ensure your script/plugin works consistently even as InDesign updates:

```javascript
// Recommended: pin a DOM version
const indesign = require("indesign");  // latest DOM
// or pin a specific version:
const indesign = require("indesign@13.0");

const app = indesign.app;
```

Available DOM versions (as of 2024): 3.0, 4.0, 5.0, 6.0, 7.0, 7.5, 8.0, 9.0, 10.0, 10.1, 10.2, 11.0, 11.2, 11.3, 11.4, 12.0, 12.1, 13.0, 13.1, 14.0, 15.0, 15.1, 16.0, 16.1, 16.2, 17.0, 18.0. For new plugins targeting InDesign 2024+, use the latest or omit the version.

---

## 4. Document model: key objects & hierarchy {#document-model}

### Application
```javascript
const { app } = require("indesign");

app.name                  // "Adobe InDesign"
app.version               // "18.5.0"
app.documents             // collection of all open documents
app.activeDocument        // the frontmost document
app.selection             // array of selected objects
app.scriptPreferences.userInteractionLevel =
  require("indesign").UserInteractionLevels.NEVER_INTERACT;  // suppress dialogs in batch
```

### Document
```javascript
const doc = app.activeDocument;

doc.name                  // filename
await doc.fullName        // Promise<string> — full path (async in UXP!)
doc.pages                 // all pages
doc.spreads               // all spreads
doc.pages.length          // page count
doc.masterSpreads         // master pages
doc.layers                // layers collection
doc.stories               // all text stories
doc.textFrames            // all text frames (top-level)
doc.allPageItems          // all page items across all pages
doc.paragraphStyles       // paragraph styles collection
doc.characterStyles       // character styles collection
doc.objectStyles          // object styles collection
doc.swatches              // colors/swatches
doc.links                 // linked files
doc.close()               // close the document
await doc.save()          // save (async)
```

### Pages and spreads
```javascript
const page = doc.pages.item(0);  // first page (0-indexed)
page.name                         // page label
page.bounds                       // [y1, x1, y2, x2] — geometric bounds
page.pageItems                   // items on this page
page.textFrames                  // text frames on this page
page.appliedMaster               // applied master page

// Iterate all pages
for (let i = 0; i < doc.pages.length; i++) {
  const page = doc.pages.item(i);
  console.log(page.name);
}
```

### Collections — always use .item(), never []
```javascript
// ✅ Correct
const firstPage = doc.pages.item(0);
const lastPage = doc.pages.item(doc.pages.length - 1);

// ❌ Wrong — does NOT work in UXP
const firstPage = doc.pages[0];

// Iterating collections
const count = doc.pages.length;
for (let i = 0; i < count; i++) {
  const page = doc.pages.item(i);
}
```

### Object comparison — never use ===
```javascript
// ✅ Correct
if (app.activeDocument.equals(doc)) { ... }

// ❌ Wrong
if (app.activeDocument === doc) { ... }
```

### Object type checking — no instanceof
```javascript
// ✅ Correct
const typeName = obj.constructor.name;  // "TextFrame", "Rectangle", etc.
if (obj.constructor.name === "TextFrame") { ... }

// ❌ Wrong
if (obj instanceof TextFrame) { ... }
```

---

## 5. Working with text {#working-with-text}

### Text frames
```javascript
const page = doc.pages.item(0);

// Add a text frame
const tf = page.textFrames.add({
  geometricBounds: [72, 72, 400, 400],  // [y1, x1, y2, x2] in points
  contents: "Hello from UXP!"
});

// Read/write content
tf.contents = "New text";

// The text story (full linked story)
const story = tf.parentStory;

// Text objects
const para = story.paragraphs.item(0);
para.contents;                  // paragraph text
para.appliedParagraphStyle;     // applied paragraph style

const word = para.words.item(0);
word.contents;

const char = para.characters.item(0);
char.pointSize = 24;
char.fillColor = doc.swatches.item("Red");
```

### Applying paragraph styles
```javascript
const style = doc.paragraphStyles.item("Heading 1");

// Apply to a paragraph
const para = tf.parentStory.paragraphs.item(0);
para.appliedParagraphStyle = style;

// Apply to all paragraphs
const story = tf.parentStory;
for (let i = 0; i < story.paragraphs.length; i++) {
  story.paragraphs.item(i).appliedParagraphStyle = style;
}
```

### Applying character styles
```javascript
const charStyle = doc.characterStyles.item("Bold Italic");
const word = tf.parentStory.words.item(0);
word.appliedCharacterStyle = charStyle;
```

### Text formatting (direct override)
```javascript
const chars = tf.parentStory.characters;
for (let i = 0; i < chars.length; i++) {
  const c = chars.item(i);
  c.pointSize = 12;
  c.leading = 16;
  c.fillColor = doc.swatches.item("Black");
  c.fontStyle = "Bold";
}
```

### Linked/threaded text frames
```javascript
// Next frame in thread
const nextFrame = tf.nextTextFrame;
// Previous frame
const prevFrame = tf.previousTextFrame;
// Thread them
tf.nextTextFrame = secondFrame;
```

### Find/change text
```javascript
app.findTextPreferences = null;  // reset
app.changeTextPreferences = null;

app.findTextPreferences.findWhat = "old text";
app.changeTextPreferences.changeTo = "new text";

doc.changeText();
```

---

## 6. Working with graphics and page items {#graphics}

### Adding shapes
```javascript
const page = doc.pages.item(0);

// Rectangle
const rect = page.rectangles.add({
  geometricBounds: [72, 72, 200, 300],
  fillColor: doc.swatches.item("Red"),
  strokeColor: doc.swatches.item("Black"),
  strokeWeight: 1
});

// Ellipse
const ellipse = page.ovals.add({
  geometricBounds: [100, 100, 200, 200]
});
```

### Placing images
```javascript
const { localFileSystem } = require("uxp").storage;

// Place image from file
const imageFile = await localFileSystem.getFileForOpening({
  types: ["jpg", "png", "tif", "pdf"]
});
const placed = page.place(imageFile.nativePath);
```

### Working with placed graphics
```javascript
const frame = doc.allPageItems.item(0);
if (frame.constructor.name === "Rectangle") {
  const graphic = frame.allGraphics.item(0);
  console.log(graphic.itemLink.name);  // linked file name
  graphic.fit(require("indesign").FitOptions.FILL_PROPORTIONALLY);
}
```

### Geometric bounds
Bounds format is always `[y1, x1, y2, x2]` in points (not pixels):
```javascript
obj.geometricBounds      // [top, left, bottom, right] — excludes stroke
obj.visibleBounds        // [top, left, bottom, right] — includes stroke
obj.geometricBounds = [72, 72, 500, 400];  // move and resize
```

### Transforming items
```javascript
const { AnchorPoint, CoordinateSpaces } = require("indesign");

rect.move(CoordinateSpaces.SPREAD_COORDINATES, [100, 100]);
rect.resize(CoordinateSpaces.SPREAD_COORDINATES, AnchorPoint.CENTER_ANCHOR,
  150, 150);
```

---

## 7. Styles {#styles}

### Paragraph styles
```javascript
// Get existing style (throws if not found — check first)
const style = doc.paragraphStyles.itemByName("Body Text");

// Check if style exists
const styleExists = doc.paragraphStyles.itemByName("Body Text").isValid;

// Create a new paragraph style
const newStyle = doc.paragraphStyles.add({
  name: "My Style",
  pointSize: 11,
  leading: 14,
  spaceBefore: 0,
  spaceAfter: 6
});

// Apply style
para.appliedParagraphStyle = style;

// Override local formatting (clear overrides)
para.clearOverrides();
```

### Character styles
```javascript
const charStyle = doc.characterStyles.add({
  name: "Highlight",
  fillColor: doc.swatches.item("Yellow")
});
```

### Object styles
```javascript
const objStyle = doc.objectStyles.itemByName("Photo Frame");
rect.appliedObjectStyle = objStyle;
```

---

## 8. Events and listeners {#events}

Events became available in InDesign v18.4+:

```javascript
const { app } = require("indesign");

// Listen for document open
app.addEventListener("afterOpen", (event) => {
  console.log("Opened:", event.target.name);
});

// Listen for before save
app.addEventListener("beforeSave", (event) => {
  console.log("About to save:", event.target.name);
});

// Listen for selection change
app.addEventListener("afterSelectionChanged", (event) => {
  const selection = app.selection;
  console.log("Selection changed, count:", selection.length);
});

// Remove listener
const handler = (event) => { ... };
app.addEventListener("afterOpen", handler);
app.removeEventListener("afterOpen", handler);
```

Common event names:
- `afterOpen`, `beforeClose`, `afterSave`, `beforeSave`, `afterSaveAs`
- `afterSelectionChanged`, `afterSelectionAttributeChanged`
- `beforePrint`, `afterPrint`
- `afterNew`, `beforeQuit`

---

## 9. Menus {#menus}

Menus became available in InDesign v18.4+:

```javascript
const { app } = require("indesign");

// Add item to the Scripts menu (or create a new submenu)
const menu = app.menus.item("$ID/Main");
const pluginMenu = menu.submenus.add({ name: "My Plugin" });
const menuItem = pluginMenu.menuItems.add({
  associatedMenuAction: app.menuActions.add({
    name: "Run My Action",
    eventListeners: [{
      eventType: "onInvoke",
      handler: () => { console.log("Menu item clicked!"); }
    }]
  })
});
```

---

## 10. Dialogs (DOM-based) {#dialogs}

InDesign has its own native dialog system via the DOM — separate from HTML UI. Use this for simple confirmation/input dialogs in commands and scripts:

```javascript
const { app } = require("indesign");

function showConfirmDialog(message) {
  const dialog = app.dialogs.add();
  const col = dialog.dialogColumns.add();

  // Static text
  const label = col.staticTexts.add();
  label.staticLabel = message;

  // Text input
  const inputRow = dialog.dialogColumns.add();
  const input = inputRow.textEditboxes.add();
  input.editContents = "Default value";
  input.minWidth = 200;

  // Dropdown
  const dropRow = dialog.dialogColumns.add();
  const dropdown = dropRow.dropdowns.add();
  dropdown.stringList = ["Option A", "Option B", "Option C"];
  dropdown.selectedIndex = 0;

  dialog.canCancel = true;
  const result = dialog.show();  // returns true if OK, false if cancelled

  if (result) {
    const userInput = input.editContents;
    const selectedOption = dropdown.stringList[dropdown.selectedIndex];
    dialog.destroy();
    return { input: userInput, option: selectedOption };
  }

  dialog.destroy();
  return null;
}
```

**Note:** If you're building a panel plugin with HTML UI, you'll typically use HTML `<dialog>` elements for dialogs, not the InDesign DOM dialog API.

HTML dialog in panel:
```html
<dialog id="myDialog">
  <sp-heading>Confirm</sp-heading>
  <sp-body>Are you sure?</sp-body>
  <footer>
    <sp-button id="cancelBtn" variant="secondary">Cancel</sp-button>
    <sp-button id="confirmBtn" variant="cta">Confirm</sp-button>
  </footer>
</dialog>
```
```javascript
const dialog = document.getElementById("myDialog");
document.appendChild(dialog);
dialog.showModal();
```

---

## 11. File operations {#file-operations}

UXP uses its own file system API — not Node.js `fs`, not CEP's `File` object.

```javascript
const { storage } = require("uxp");
const fs = storage.localFileSystem;
const { formats } = storage;

// Open file picker
const file = await fs.getFileForOpening({
  types: ["txt", "json", "csv"],
  initialLocation: await fs.getTemporaryFolder()
});
if (!file) return;  // user cancelled

// Read file
const text = await file.read({ format: formats.utf8 });
const data = JSON.parse(text);

// Save file picker
const saveFile = await fs.getFileForSaving("output.txt", {
  types: ["txt"]
});
if (!saveFile) return;

// Write file
await saveFile.write("Hello World", { format: formats.utf8 });

// Plugin's own folders (no permission needed)
const pluginFolder = await fs.getPluginFolder();
const dataFolder = await fs.getDataFolder();
const tempFolder = await fs.getTemporaryFolder();

// Create a file in plugin's data folder
const dataFile = await dataFolder.createFile("settings.json", { overwrite: true });
await dataFile.write(JSON.stringify({ key: "value" }), { format: formats.utf8 });

// List folder contents
const entries = await dataFolder.getEntries();
for (const entry of entries) {
  console.log(entry.name, entry.isFolder ? "(folder)" : "(file)");
}
```

**Manifest permissions required:**
```json
"requiredPermissions": {
  "localFileSystem": "request"  // for file pickers
  // "localFileSystem": "fullAccess"  // for arbitrary path access
}
```

---

## 12. Network requests {#network}

UXP supports `fetch()` natively. Add the domain to your manifest first:

```json
"requiredPermissions": {
  "network": {
    "domains": ["https://api.example.com"]
  }
}
```

```javascript
// GET request
const response = await fetch("https://api.example.com/data");
const data = await response.json();

// POST request with JSON body
const result = await fetch("https://api.example.com/save", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "My Document", pages: 10 })
});

// Error handling
try {
  const res = await fetch("https://api.example.com/items");
  if (!res.ok) throw new Error(`HTTP error: ${res.status}`);
  const items = await res.json();
} catch (err) {
  console.error("Network error:", err.message);
}
```

**Note:** InDesign has no protocol restrictions (unlike Photoshop) — HTTP and HTTPS both work. But always declare domains in the manifest. Using `"domains": "all"` works but will display a warning to users in future versions.

---

## 13. Storage (plugin data persistence) {#storage}

For storing small amounts of plugin settings/preferences that persist between sessions:

```javascript
const { storage } = require("uxp");
const { localFileSystem, formats } = storage;

// Use the plugin's data folder for persistent settings
async function saveSettings(settings) {
  const dataFolder = await localFileSystem.getDataFolder();
  const file = await dataFolder.createFile("settings.json", { overwrite: true });
  await file.write(JSON.stringify(settings), { format: formats.utf8 });
}

async function loadSettings() {
  try {
    const dataFolder = await localFileSystem.getDataFolder();
    const file = await dataFolder.getEntry("settings.json");
    const text = await file.read({ format: formats.utf8 });
    return JSON.parse(text);
  } catch (e) {
    return null;  // file doesn't exist yet
  }
}
```

**Plugin storage locations (no permission needed):**
- `plugin://` — plugin installation folder (read-only)
- `plugin-data://` — persistent per-plugin data folder
- `plugin-temp://` — temporary folder (cleared periodically)

---

## 14. Scripts (.idjs) {#scripts}

UXP scripts are single `.idjs` files that run directly in InDesign without install.

**Script entry points:** Scripts run top-to-bottom, and support `await` at the top level (global await):
```javascript
// my-script.idjs
const indesign = require("indesign");
const { app } = indesign;

// Top-level await works in .idjs files
const doc = app.activeDocument;
const pageCount = doc.pages.length;

// Show a dialog and wait for user
const result = await showMyDialog();
if (!result) {
  console.log("User cancelled.");
  // Script simply ends here
}

console.log(`Done. Processed ${pageCount} pages.`);

async function showMyDialog() {
  const dialog = app.dialogs.add();
  const col = dialog.dialogColumns.add();
  const label = col.staticTexts.add();
  label.staticLabel = "Process all pages?";
  dialog.canCancel = true;
  const ok = dialog.show();
  dialog.destroy();
  return ok;
}
```

**Script args (passing parameters to scripts):**
```javascript
const { script } = require("uxp");
// Read args passed to script
const args = script.args;  // array of strings
const targetPage = parseInt(args[0]) || 1;

// Set result (return value)
script.setResult("Done processing " + doc.name);
```

**Where to put scripts:**
- `~/Documents/Adobe Scripts/` — auto-discovered
- Any location via Scripts panel → double-click to install
- Via `app.doScript()` from a plugin: `app.doScript(scriptText, indesign.ScriptLanguage.UXPSCRIPT)`

**Script vs plugin decision:**
- No persistent UI needed, runs once → Script
- Need a side panel that stays open → Plugin
- Internal automation, CI/batch → Script
- User-facing tool → Plugin

---

## 15. React plugin setup {#react-setup}

For complex plugins with lots of UI state, React is the best choice.

### Project structure
```
my-react-plugin/
├── manifest.json       ← goes into dist/ via CopyPlugin
├── package.json
├── webpack.config.js
├── src/
│   ├── index.jsx       ← entrypoints.setup() here
│   ├── controllers/
│   │   └── PanelController.jsx
│   └── panels/
│       └── MainPanel.jsx
└── dist/               ← built output — load this folder in UDT
    ├── manifest.json   ← copied from root
    ├── index.html
    └── index.js        ← webpack bundle
```

### package.json
```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "scripts": {
    "build": "webpack --mode development",
    "watch": "webpack --mode development --watch"
  },
  "devDependencies": {
    "@babel/core": "^7.22.0",
    "@babel/plugin-transform-react-jsx": "^7.22.0",
    "@babel/proposal-object-rest-spread": "^7.20.0",
    "@babel/plugin-syntax-class-properties": "^7.12.0",
    "babel-loader": "^9.0.0",
    "clean-webpack-plugin": "^4.0.0",
    "copy-webpack-plugin": "^11.0.0",
    "css-loader": "^6.0.0",
    "style-loader": "^3.0.0",
    "webpack": "^5.80.0",
    "webpack-cli": "^5.0.0"
  },
  "dependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  }
}
```

### webpack.config.js
```javascript
const path = require("path");
const CopyPlugin = require("copy-webpack-plugin");

module.exports = {
  entry: "./src/index.jsx",
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "index.js"
  },
  devtool: "source-map",
  // CRITICAL: mark uxp and indesign as externals — they come from the host app
  externals: {
    uxp: "commonjs2 uxp",
    indesign: "commonjs2 indesign",
    os: "commonjs2 os"
  },
  resolve: { extensions: [".js", ".jsx"] },
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        exclude: /node_modules/,
        loader: "babel-loader",
        options: {
          plugins: [
            "@babel/transform-react-jsx",
            "@babel/proposal-object-rest-spread",
            "@babel/plugin-syntax-class-properties"
          ]
        }
      },
      { test: /\.css$/, use: ["style-loader", "css-loader"] }
    ]
  },
  plugins: [
    new CopyPlugin({
      patterns: [
        { from: "manifest.json", to: "manifest.json" },
        { from: "icons", to: "icons" }
      ]
    })
  ]
};
```

### src/index.jsx — main entry
```jsx
import React from "react";
import { entrypoints } from "uxp";
import { PanelController } from "./controllers/PanelController.jsx";
import { MainPanel } from "./panels/MainPanel.jsx";

const mainPanelController = new PanelController(() => <MainPanel />, {
  id: "mainPanel"
});

entrypoints.setup({
  plugin: {
    create(plugin) { console.log("Plugin created", plugin); },
    destroy()      { console.log("Plugin destroyed"); }
  },
  panels: {
    mainPanel: mainPanelController
  }
});
```

### PanelController pattern
```jsx
import ReactDOM from "react-dom";

export class PanelController {
  constructor(Component, { id } = {}) {
    this._id = id;
    this._root = null;
    this._attachment = null;
    this._Component = Component;
    ["create", "show", "hide", "destroy"].forEach(fn => this[fn] = this[fn].bind(this));
  }
  create() {
    this._root = document.createElement("div");
    this._root.style.height = "100vh";
    this._root.style.overflow = "auto";
    ReactDOM.render(this._Component({ panel: this }), this._root);
    return this._root;
  }
  show(event) {
    if (!this._root) this.create();
    this._attachment = event;
    this._attachment.appendChild(this._root);
  }
  hide() {
    if (this._attachment && this._root) {
      this._attachment.removeChild(this._root);
      this._attachment = null;
    }
  }
  destroy() {}
}
```

### MainPanel.jsx — example panel
```jsx
import React, { useState } from "react";

export function MainPanel() {
  const [status, setStatus] = useState("");

  async function handleRun() {
    try {
      const { app } = require("indesign");
      const doc = app.activeDocument;
      setStatus(`Processing: ${doc.name}`);

      const count = doc.pages.length;
      setStatus(`Done! Found ${count} pages.`);
    } catch (err) {
      setStatus(`Error: ${err.message}`);
    }
  }

  return (
    <div style={{ padding: 16 }}>
      <h3>My Plugin</h3>
      <button onClick={handleRun}>Run</button>
      {status && <p>{status}</p>}
    </div>
  );
}
```

**Note:** `require("indesign")` must be called inside the event handler, not at the top of the file, when using webpack — because it's an external loaded at runtime.

---

## 16. Known limitations and gotchas {#gotchas}

### InDesign-specific
- **Collections**: always use `.item(N)`, never `[N]`
- **Object equality**: always use `.equals()`, never `===`
- **`instanceof`**: not supported for InDesign DOM objects — use `.constructor.name`
- **`app.activeDocument.fullName`**: returns a `Promise` in v18.5+ (was a string in ExtendScript) — always `await` it
- **`app.getPluginFolder()`**: removed in v18.4 for scripts — use `localFileSystem.getDataFolder()` instead
- **Enumerators** (like `UserInteractionLevels`) are only accessible via `require("indesign")`, not globally

### UXP CSS engine
- No `display: grid` — use flexbox
- No `gap` in flexbox — use `margin`
- No `float`
- No `aspect-ratio`
- CSS animations (`transition`, `@keyframes`) are unreliable — avoid
- `CSS.setProperty('--var', value)` for custom properties — not supported

### UXP JavaScript
- `DOMParser` — not available (`new DOMParser()` throws)
- `TextDecoder` — not available
- `structuredClone()` — not available
- `window.matchMedia()` — not available; for theme: use `document.theme.onUpdated.addListener()`
- `XPathResult`, `document.evaluate()` — not available
- `navigator.language` — returns undefined; use `require("uxp").host.locale`
- Custom property assignment via `element.style.setProperty('--prop', val)` — not supported

### Async patterns
- UXP InDesign operations are often async — always use `async/await`
- Promise that never resolves = almost always a missing permission in manifest
- Global await is allowed in `.idjs` scripts but not in plugin entry files
- `app.activeScript` is now async and returns a Promise in v18.4+

### Manifest
- Changing `manifest.json` requires **full unload + reload** — Watch won't pick it up
- Using `"domains": "all"` for network will trigger user consent warnings in future versions

---

## 17. ExtendScript → UXP migration cheatsheet {#migration}

| ExtendScript (JSX) | UXP (.idjs / plugin) |
|---|---|
| `app` (global) | `const { app } = require("indesign")` |
| `app.activeDocument.fullName` | `await app.activeDocument.fullName` |
| `doc.pages[0]` | `doc.pages.item(0)` |
| `story.paragraphs[0]` | `story.paragraphs.item(0)` |
| `objA === objB` | `objA.equals(objB)` |
| `obj instanceof TextFrame` | `obj.constructor.name === "TextFrame"` |
| `$.writeln("msg")` | `console.log("msg")` |
| `File.openDialog("Select file")` | `await fs.getFileForOpening({...})` |
| `app.scriptArgs.getValue("x")` | `require("uxp").script.args[0]` |
| `app.activeDocument.constructor.name` | same in v18.4+ |
| ECMAScript 3 | ECMAScript 6+ (V8 engine) |
| Synchronous only | async/await supported, global await in scripts |
| `$.locale` | `require("uxp").host.locale` |

**Key mindset shift:** In UXP, InDesign DOM access is now a module, many operations are async Promises, and collections use `.item()`. The rest of the DOM API (pages, text frames, styles, etc.) is largely the same shape as ExtendScript — just accessed differently.
