# RR-MS-Tower-Stage

An AutoCAD 2027 .NET plugin (C#, `net10.0-windows`, WinForms) that draws steel column blocks for MS (mild-steel) parking towers, positioned from tower-combination CTC data. All drawing units are mm.

The plugin exposes four commands, all registered in [`PluginEntry.cs`](PluginEntry.cs):

| Command | Purpose |
|---|---|
| `RR-MS-Tower-Stage` | Opens the two-page input wizard and inserts the full tower drawing (Front/Rear elevations, per-column Line elevations, Plan views) as one master block. |
| `RR-STAGE-Editor` | Resizes an existing stage's length after the tower has been drawn. |
| `RR-Explode` | Bakes splice holes into static per-column blocks and explodes the master wrapper, once stage editing is finished. |
| `RR-Bracing` | Standalone bracing-block insertion command, independent of the tower pipeline. |

For full behavioral detail (block formulas, gating rules, dynamic block parameters), see [`docs/project/`](docs/project/) — start with [`01-ai-workflow.md`](docs/project/01-ai-workflow.md) and [`04-repository-layout.md`](docs/project/04-repository-layout.md).

## Repository Layout

```
RR-MS-Tower-Stage.csproj   — SDK-style project, AutoCAD 2027 references
PluginEntry.cs             — command entry points
TowerStageInput*.cs        — input wizard (data model + UI, split into partial-class files)
TowerStageGeometryBuilder*.cs — drawing engine (split into partial-class files by feature)
DrawingHelper.cs / DynamicBlockHelper.cs — shared drawing + dynamic-block primitives
*Table.cs                  — hardcoded lookup tables (ground truth in Data\)
build-testload.ps1         — timestamped test-build script
Data\                      — source files; ground truth for the lookup tables
docs\project\              — numbered reference docs (architecture, conventions, block reference)
SpecSheet\                 — one file per content-library block family (spec, formula, rationale)
ChangeLog\YYYY-MM-DD.md    — one file per calendar day, append-only build log
TestLoad\, LatestBuild\, bin\, obj\ — build artifacts (disposable / generated)
```

See [`docs/project/04-repository-layout.md`](docs/project/04-repository-layout.md) for the complete, maintained file-by-file breakdown and function index.

## Prerequisites

- Windows with AutoCAD 2027 installed at `C:\Program Files\Autodesk\AutoCAD 2027\` (the project references `accoremgd.dll`, `acdbmgd.dll`, `acmgd.dll` from that path).
- .NET SDK supporting `net10.0-windows`.
- PowerShell.

## Standard Operating Procedure

### 1. Build for testing

Run the build script from the project root:

```powershell
.\build-testload.ps1
```

This produces a timestamped assembly at `TestLoad\<yyyyMMdd_HHmmss>\RRMSTowerStage_<stamp>.dll`. A fresh timestamp is required every time because .NET assemblies never unload from a running AutoCAD session — a same-named DLL is silently ignored on re-`NETLOAD`.

### 2. Load and test in AutoCAD

1. Open AutoCAD 2027.
2. Run `NETLOAD` and select the newly built `.dll` from the `TestLoad\<stamp>\` folder.
3. The most recently loaded build owns the `RR-MS-Tower-Stage` command registration.
4. Run `RR-MS-Tower-Stage`, `RR-STAGE-Editor`, `RR-Explode`, or `RR-Bracing` as needed and verify the drawing output visually.

### 3. Checkpoint a verified build ("COPY BUILD")

Once a `TestLoad\<stamp>\` build is confirmed working, snapshot it into a matching `LatestBuild\<stamp>\` folder: copy every root `*.cs` file, the current `ChangeLog\`, `project.md`, and `STAGING_REQUIREMENTS.md`, plus that build's `.dll`/`.pdb`/`.deps.json`. This is a pure filesystem snapshot — no rebuild involved.

### 4. Documentation requirements for any change

Every change that adds or alters drawing behavior must update, in the same cycle as the code change:

- **`ChangeLog\YYYY-MM-DD.md`** — one file per calendar day, created fresh on the first major change of the day and appended to afterward. Never delete or cap retention. If a new entry supersedes a prior one, mark the old entry `(superseded)` rather than removing it.
- **`docs\project\09-drawing-generation.md`** — new block name or formula gets a `§7` bullet with the verbatim requirement, the formula, and its gating condition.
- **`docs\project\12-block-reference.md`** — new row for any new content-library block.
- **`docs\project\13-variables.md`** — new line for any new `const`/`static readonly`/`TowerStageInput` field.
- **`SpecSheet\`** — new entry per `SpecSheet\README.md` and `SpecSheet\_TEMPLATE.md` conventions: identity, spec, why the logic is valid, how it's performed, whether `RR-Explode` acts on it, and known limitations.
- **`docs\project\04-repository-layout.md`** — update the file/function index whenever a function or file is added, removed, renamed, or moved.

Check the relevant SpecSheet first before reading source code to answer a block question; fall back to source only if the SpecSheet is missing or outdated.

### 5. Engineering conventions

- **Small files only.** When a file grows large, split it into `partial class` files along existing logical/feature boundaries. Each split must be a pure relocation (zero logic change) with a clean build verified immediately after.
- **New requirement → new functions/files**, never folded into an existing function, even if structurally similar. Existing functions are only touched for an explicit bug-fix or change request against that existing feature.
- **Attributes are sacred.** Never erase or write block attributes outside the two sanctioned paths (`AttachAttributes`, `SetAttributeValue` at its two specified call sites) plus `RR-Explode` re-attach.
- **`RR-Explode` must stay tower-condition-aware.** Any `Propagate*` function's bake/mirror/copy action must match the same Close/Open Tower, combo, and Tower-Type conditions as the live insertion it propagates.
- **Debug instrumentation** must use the copyable `DebugAlert()` modal (`DynamicBlockHelper.cs`), never raw command-line text, and must be removed once the user confirms the associated feature works as intended.
- Ground truth for lookup tables is the raw files in `Data\` — if a table value looks wrong, check against the source file, not the C# comment.

Full detail and rationale for every rule above lives in [`docs/project/01-ai-workflow.md`](docs/project/01-ai-workflow.md); read it in full before making structural changes.

## Status

No automated test suite and no git version control is currently in use for this project; `ChangeLog\` and `docs\project\` serve as the change history and architecture record.
