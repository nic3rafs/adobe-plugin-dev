# UXP Common Patterns

Shared patterns that work across all Adobe UXP apps (InDesign, Photoshop, Illustrator, etc.).

## Table of Contents
1. [UXP module reference](#uxp-modules)
2. [File system API](#filesystem)
3. [Network (fetch)](#network)
4. [Persistent storage (plugin data)](#persistent-storage)
5. [Clipboard](#clipboard)
6. [Host info and locale](#host-info)
7. [React + webpack setup](#react-setup)
8. [Spectrum Web Components (SWC)](#swc)
9. [WebView](#webview)
10. [Inter-plugin communication (IPC)](#ipc)
11. [Debugging with UDT](#debugging)
12. [Packaging and distribution](#distribution)
13. [Plugin ID and developer registration](#plugin-id)

---

## 1. UXP module reference {#uxp-modules}

```javascript
const uxp = require("uxp");

// Sub-modules
const { storage, clipboard, shell, host, script, entrypoints } = uxp;

// Or import individually
const { entrypoints } = require("uxp");
const { storage } = require("uxp");
const { clipboard } = require("uxp");
```

All UXP modules are available via `require("uxp")`. The host app module (`indesign`, `photoshop`, etc.) is separate.

---

## 2. File system API {#filesystem}

UXP uses its own async file API — not Node.js `fs`.

```javascript
const { storage } = require("uxp");
const fs = storage.localFileSystem;
const { formats } = storage;

// ── File Pickers ──────────────────────────────────────

// Open file picker (returns null if user cancels)
const file = await fs.getFileForOpening({
  types: ["txt", "json", "csv"],       // allowed extensions
  allowMultiple: false,                 // or true for multi-select
  initialLocation: await fs.getTemporaryFolder()
});

// Save file picker
const saveFile = await fs.getFileForSaving("output.json", {
  types: ["json"]
});

// Folder picker
const folder = await fs.getFolder();

// ── Read / Write ──────────────────────────────────────

// Read text
const text = await file.read({ format: formats.utf8 });

// Read binary
const buffer = await file.read({ format: formats.binary });

// Write text
await saveFile.write("Hello", { format: formats.utf8 });

// Write binary (ArrayBuffer)
await saveFile.write(arrayBuffer, { format: formats.binary });

// Append text
await saveFile.write("more text", { format: formats.utf8, append: true });

// ── Plugin Folders (no permission needed) ────────────

const pluginFolder = await fs.getPluginFolder();  // read-only, your plugin's install dir
const dataFolder   = await fs.getDataFolder();    // persistent, writable
const tempFolder   = await fs.getTemporaryFolder();  // writable, not guaranteed to persist

// ── Folder operations ────────────────────────────────

const entries = await dataFolder.getEntries();    // [{name, isFolder, isFile, ...}]
const subFolder = await dataFolder.createFolder("my-subfolder");
const newFile = await dataFolder.createFile("settings.json", { overwrite: true });

// Check if entry exists
const exists = await dataFolder.getEntry("settings.json").then(() => true).catch(() => false);

// Delete
await file.delete();
```

**Manifest permissions:**
```json
"requiredPermissions": {
  "localFileSystem": "plugin"     // plugin folders only (safest — default)
  // "localFileSystem": "request" // + file/folder pickers
  // "localFileSystem": "fullAccess" // + arbitrary paths (use sparingly)
}
```

---

## 3. Network (fetch) {#network}

UXP supports the standard `fetch()` API:

```javascript
// Manifest must declare the domain first:
// "requiredPermissions": { "network": { "domains": ["https://api.example.com"] } }

// GET
const response = await fetch("https://api.example.com/data");
if (!response.ok) throw new Error(`HTTP ${response.status}`);
const data = await response.json();

// POST JSON
const res = await fetch("https://api.example.com/submit", {
  method: "POST",
  headers: { "Content-Type": "application/json", "Authorization": "Bearer TOKEN" },
  body: JSON.stringify({ key: "value" })
});

// Upload file (multipart)
const formData = new FormData();
formData.append("file", fileBlob, "filename.pdf");
await fetch("https://api.example.com/upload", { method: "POST", body: formData });

// Download binary
const imgRes = await fetch("https://example.com/image.png");
const arrayBuffer = await imgRes.arrayBuffer();

// With timeout (UXP doesn't support AbortController signal, implement manually)
const withTimeout = (promise, ms) =>
  Promise.race([promise, new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), ms)
  )]);

const data = await withTimeout(fetch("https://api.example.com/slow"), 10000);
```

**Manifest:**
```json
"requiredPermissions": {
  "network": {
    "domains": [
      "https://api.myservice.com",
      "https://cdn.myservice.com"
    ]
    // or: "domains": "all"  — works but generates future user-consent warnings
  }
}
```

---

## 4. Persistent storage (plugin data) {#persistent-storage}

Two approaches: file-based (most flexible) or UXP's built-in key-value store (simple).

### File-based settings (recommended)
```javascript
const { storage } = require("uxp");
const { localFileSystem, formats } = storage;

const SETTINGS_FILE = "settings.json";

async function saveSettings(data) {
  const folder = await localFileSystem.getDataFolder();
  const file = await folder.createFile(SETTINGS_FILE, { overwrite: true });
  await file.write(JSON.stringify(data, null, 2), { format: formats.utf8 });
}

async function loadSettings(defaults = {}) {
  try {
    const folder = await localFileSystem.getDataFolder();
    const file = await folder.getEntry(SETTINGS_FILE);
    const text = await file.read({ format: formats.utf8 });
    return { ...defaults, ...JSON.parse(text) };
  } catch {
    return defaults;  // first run — no settings file yet
  }
}

// Usage
const settings = await loadSettings({ theme: "dark", fontSize: 12 });
await saveSettings({ ...settings, fontSize: 14 });
```

---

## 5. Clipboard {#clipboard}

```javascript
const { clipboard } = require("uxp");

// Write text to clipboard
await clipboard.setContent({ "text/plain": "Hello clipboard!" });

// Read text from clipboard
const content = await clipboard.getContent();
const text = content["text/plain"];

// Check available types
const types = Object.keys(content);
```

**Manifest:**
```json
"requiredPermissions": {
  "clipboard": "readAndWrite"   // or "read"
}
```

---

## 6. Host info and locale {#host-info}

```javascript
const { host } = require("uxp");

host.name           // "InDesign" / "Photoshop" / etc.
host.version        // "18.5.0"
host.locale         // "en_US" — use this instead of navigator.language
host.uiLocale       // UI locale (may differ from document locale)
host.platform       // "win" | "mac"
```

Use `host.name` to write code that behaves differently per app:
```javascript
const { host } = require("uxp");

if (host.name === "InDesign") {
  const { app } = require("indesign");
  // InDesign-specific code
} else if (host.name === "Photoshop") {
  const { app } = require("photoshop");
  // Photoshop-specific code
}
```

---

## 7. React + webpack setup {#react-setup}

The critical thing with React in UXP plugins is that `uxp`, `indesign` (and other host modules) must be **externals** — they're provided by the host app at runtime, not bundled by webpack.

### Key webpack rule
```javascript
// webpack.config.js — ALWAYS include these externals
externals: {
  uxp: "commonjs2 uxp",
  indesign: "commonjs2 indesign",   // InDesign
  photoshop: "commonjs2 photoshop", // Photoshop
  illustrator: "commonjs2 illustrator", // Illustrator (if needed)
  os: "commonjs2 os"
}
```

### Using host modules in React components
Because `indesign` is an external, `require("indesign")` is fine inside component methods and event handlers. Don't call it at module top-level if you're writing cross-host code (it would throw in Photoshop):

```jsx
// ✅ Safe — inside an event handler
function MyComponent() {
  async function handleClick() {
    const { app } = require("indesign");
    const doc = app.activeDocument;
    console.log(doc.name);
  }
  return <button onClick={handleClick}>Run</button>;
}

// ❌ Risky for cross-host plugins — throws if not InDesign
import { app } from "indesign";  // don't do this at top level for multi-host
```

### index.html for React panel
The HTML file just needs to load the built bundle. webpack/babel handles JSX compilation:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <script src="index.js"></script>
</head>
<body>
  <div id="root"></div>
</body>
</html>
```

Then in `src/index.jsx`:
```jsx
import React from "react";
import ReactDOM from "react-dom";
import { entrypoints } from "uxp";
import App from "./App.jsx";

entrypoints.setup({
  panels: {
    myPanel: {
      show({ node }) {
        ReactDOM.render(<App />, node);
      },
      hide() {},
      destroy() {}
    }
  }
});
```

This is a simpler alternative to the PanelController class — mount React directly into the `node` that UXP provides to the `show()` callback.

---

## 8. Spectrum Web Components (SWC) {#swc}

Adobe's design system, fully supported in UXP plugins (stable as of InDesign 18.4+).

**Manifest requirement:**
```json
"featureFlags": {
  "enableSWCSupport": true
}
```

**Install:**
```bash
# Using yarn (recommended by Adobe)
yarn add @spectrum-web-components/theme
yarn add @swc-uxp-wrappers/utils
yarn add @swc-uxp-wrappers/button
yarn add @swc-uxp-wrappers/textfield
yarn add @swc-uxp-wrappers/checkbox
yarn add @swc-uxp-wrappers/dropdown

# npm
npm install @spectrum-web-components/theme @swc-uxp-wrappers/button
```

**Usage in HTML:**
```html
<!-- Wrap everything in sp-theme -->
<sp-theme theme="spectrum" color="light" scale="medium" dir="ltr">
  <sp-button variant="primary" id="myBtn">Click Me</sp-button>
  <sp-textfield placeholder="Enter text..." id="myInput"></sp-textfield>
  <sp-checkbox id="myCheck">Enable feature</sp-checkbox>
</sp-theme>
```

**Usage in JS (ES module import):**
```javascript
import "@spectrum-web-components/theme/sp-theme.js";
import "@spectrum-web-components/theme/src/themes.js";
import "@swc-uxp-wrappers/button/sp-button.js";
import "@swc-uxp-wrappers/textfield/sp-textfield.js";
```

**Built-in Spectrum UXP tags (no install needed):**
`sp-button`, `sp-textfield`, `sp-checkbox`, `sp-radio`, `sp-dropdown`, `sp-slider`,
`sp-heading`, `sp-body`, `sp-label`, `sp-divider`, `sp-link`, `sp-progressbar`, `sp-icon`

These are UXP's built-in Spectrum-flavored elements and don't require any npm packages.

---

## 9. WebView {#webview}

Embed a full web page inside a plugin panel. Good for complex UIs, external web apps, or content loaded from a server.

**Manifest:**
```json
"requiredPermissions": {
  "webview": {
    "allow": "yes",
    "domains": ["https://my-app.example.com", "http://127.0.0.1:8080"],
    "enableMessageBridge": "localAndRemote"  // enables postMessage()
  }
}
```

**HTML:**
```html
<webview id="myWebview" src="https://my-app.example.com" width="100%" height="400px"></webview>
```

**Plugin → WebView messaging:**
```javascript
const webview = document.getElementById("myWebview");
webview.postMessage("hello from plugin");

// Events
webview.addEventListener("loadstart", (e) => console.log("loading:", e.url));
webview.addEventListener("loadstop",  (e) => console.log("loaded:",  e.url));
webview.addEventListener("loaderror", (e) => console.error("error:", e.url));
```

**WebView → Plugin messaging:**
```javascript
window.addEventListener("message", (e) => {
  console.log("From webview:", e.data);
});
```

---

## 10. Inter-plugin communication (IPC) {#ipc}

Plugins can communicate with each other using UXP's IPC module.

**Manifest:**
```json
"requiredPermissions": {
  "ipc": { "enablePluginCommunication": true }
}
```

```javascript
const { ipc } = require("uxp");

// Send message to another plugin by its ID
ipc.sendMessage("com.other.plugin.id", "myEvent", { data: "hello" });

// Receive messages from other plugins
ipc.on("myEvent", (sender, data) => {
  console.log("From:", sender, "Data:", data);
});
```

---

## 11. Debugging with UDT {#debugging}

UXP Developer Tools (UDT) is the primary debugging environment.

### Install UDT
1. Open **Creative Cloud Desktop**
2. Go to **All Apps** tab
3. Search for **UXP Developer Tools**
4. Click **Install**

### Daily workflow
```
UDT → Add Plugin → select plugin folder → Load
```

- **Load** — loads plugin into the host app
- **Reload** — hot-reloads (for file changes; NOT manifest changes)
- **Watch** — auto-reloads on file save (shows a file-watcher icon)
- **Debug** — opens a Chrome DevTools-like debugger window
- **Package** — builds a `.ccx` file for distribution

### In the debugger you can
- View `console.log()` / `console.error()` output
- Set breakpoints
- Inspect the DOM (plugin HTML structure)
- Inspect network requests
- Step through async code

### CLI alternative (devtools-cli)
```bash
npm install -g @adobe/uxp-devtools-cli
uxp plugin load          # load from current directory
uxp plugin reload        # reload
uxp plugin debug         # open debugger
```

### Useful debugging patterns
```javascript
// Log InDesign objects (use JSON.stringify carefully — deep objects can be huge)
const doc = app.activeDocument;
console.log("Doc:", doc.name, "Pages:", doc.pages.length);

// Log selection
console.log("Selection:", app.selection.map(s => s.constructor.name));

// Catch and log errors explicitly (UXP sometimes swallows errors)
try {
  const result = await myAsyncOperation();
} catch (err) {
  console.error("Error in myAsyncOperation:", err.message, err.stack);
}
```

---

## 12. Packaging and distribution {#distribution}

### Package your plugin
```
UDT → select plugin → Actions → Package
```
Choose output directory → UDT creates `PluginName.ccx`.

The `.ccx` file is a zip archive containing your built plugin. Users install it by double-clicking (Creative Cloud app handles it).

### Pre-distribution checklist
- [ ] Replace placeholder `id` in manifest with your real Developer Distribution ID
- [ ] Set correct `version` in manifest (semver, e.g. `"1.2.0"`)
- [ ] Set `"minVersion"` accurately in `host`
- [ ] Remove `console.log` statements or add a debug flag
- [ ] Test on both macOS and Windows if possible
- [ ] Test with a clean user profile (no loaded plugins except yours)
- [ ] Verify all manifest permissions are the minimum required
- [ ] Check that icons exist at all referenced paths

### Distribution options

**1. Direct distribution (.ccx file)**
Share the `.ccx` file. Users double-click to install via Creative Cloud.
Good for: internal tools, beta testing, small teams.

**2. Enterprise distribution**
For companies: deploy via Creative Cloud for Enterprise managed config.
Good for: corporate tools deployed to many users.

**3. Creative Cloud Marketplace (public)**
- Submit via [Adobe Developer Distribution Portal](https://developer.adobe.com/developer-distribution/)
- Requires review process (1-3 weeks typically)
- Gets listed in the Plugins marketplace inside Creative Cloud
- Maximum reach — users can install directly from the app

---

## 13. Plugin ID and developer registration {#plugin-id}

Every published plugin needs a unique ID from Adobe.

1. Go to [developer.adobe.com/developer-distribution/](https://developer.adobe.com/developer-distribution/)
2. Sign in with your Adobe ID
3. Create a **New Plugin Listing**
4. Copy the generated **Plugin ID** (looks like `com.yourname.pluginname`)
5. Paste it into `manifest.json` → `"id"` field
6. This ID is permanent — don't change it after publishing

**For development only** you can use any reverse-domain string. But you MUST get a real ID before distributing.

```json
{
  "id": "com.mycompany.myplugin",   ← replace with real ID from portal
  ...
}
```
