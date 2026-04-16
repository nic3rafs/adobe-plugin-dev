# adobe-plugin-dev

Claude Code skill for building Adobe Creative Cloud plugins with **UXP** (Unified Extensibility Platform).

Scaffold, build, debug, and ship InDesign plugins (extensible to Photoshop, Illustrator, Premiere Pro, After Effects) — Claude handles the boilerplate, DOM differences, and tricky UXP gotchas.

## What it covers

- UXP plugin structure (manifest, entrypoints, panels vs. commands vs. scripts)
- InDesign DOM + scripting APIs (full reference included)
- Shared UXP patterns across CC apps (`uxp-common.md`)
- Local dev + UDT debugging workflow
- Packaging + distribution via Creative Cloud Marketplace

## Install (Claude Code)

```
/plugin marketplace add nic3rafs/adobe-plugin-dev
/plugin install adobe-plugin-dev@adobe-plugin-dev
```

## Usage

Ask Claude anything like:

- "Build an InDesign plugin that renumbers paragraphs"
- "Create a UXP panel for Photoshop that exports layers to PNG"
- "Migrate my CEP extension to UXP"
- "Add a script to auto-apply paragraph styles in InDesign"

The skill auto-activates on UXP / Adobe-plugin mentions.

## Structure

```
.
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
└── skills/
    └── adobe-plugin-dev/
        ├── SKILL.md
        └── references/
            ├── indesign.md
            └── uxp-common.md
```

## Roadmap

- Photoshop reference
- Illustrator reference
- Premiere Pro reference
- After Effects reference

Contributions welcome.

## License

MIT — see [LICENSE](./LICENSE).
