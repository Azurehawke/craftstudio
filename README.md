# CraftStudio (working title)

Drafting and plan markup in one app: CADCraft's drawing tools and RevuCraft's PDF markup and
takeoff, with the toolbars changing by mode, for Windows and Android tablets.

**Status: planning.** No app code yet. The two apps are being refined first; see
[docs/CONTEXT.md](docs/CONTEXT.md) for the plan, the architecture decision and the open questions.

## What's here

- `upstream/pdfcraft`: RevuCraft (Azurehawke/pdfcraft, branch `revucraft`), as a git submodule.
- `upstream/cadcraft`: CADCraft (Azurehawke/cadcraft), as a git submodule.
- `docs/CONTEXT.md`: the plan.

Clone with the submodules:

```
git clone --recurse-submodules https://github.com/Azurehawke/craftstudio
```

Licensed MIT OR Apache-2.0, like both upstream projects. Bluebeam, Revu, Autodesk, AutoCAD and
Affinity are trademarks of their owners; this project is not affiliated with them.
