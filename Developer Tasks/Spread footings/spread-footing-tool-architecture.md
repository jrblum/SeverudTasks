---
tags: [project, structural, revit, python]
status: design
version: 2
---

# Spread Footing Tool: Schema, Module Layout and Deployment

## Scope (locked in)

- **Rectangular footings only, one column per footing.** Anything else is flagged as `out_of_scope` and the engineer designs it by hand. The tool exists to take the easy footings off the engineers' plates.
- **Codes:** ACI 318-19 or 318-14 (switchable), ASCE 7-16 load combinations. **ASD for bearing, LRFD for strength.**
- **Column loads** come from parameters on the column, written by another in-house plugin.
- **Runs locally on each engineer's machine**, started from a button in Revit.
- **Revit is for modeling, not design.** The add-in exports, launches, and writes back a summary. No design logic lives in C#.

### Scope gate (what counts as "easy")

The engine should return one of `pass`, `fail`, or `out_of_scope` with reasons. Suggested `out_of_scope` triggers:

- not exactly one column matched to the footing
- resultant outside the kern under any ASD service combination (partial contact: engineer reviews)
- net uplift under any combination (e.g. 0.6D + 0.6W, 0.6D + 0.7E)
- missing load cases or missing parameters
- footing or column dimensions outside configured limits

This keeps the engine small and avoids the tool producing a plausible answer for a case it was never meant to handle. Partial-contact bearing can be added later if you want it.

---

## Conventions

- **Units in the exchange files:** lengths in **inches**, forces in **kip**, moments in **kip-in**, concrete strength in **ksi**, soil pressure in **ksf**. The add-in converts using the Revit unit API (`UnitUtils.ConvertFromInternalUnits` with an explicit unit id), never by assuming Revit's internal units. Units are also written into the file.
- **Coordinates:** global project coordinates (not shared coordinates).
- **Sign convention:** P is positive downward (compression). Mx and My follow the right-hand rule about global X and Y. Fix this once and document it.
- **Keys:** `Element.UniqueId` (string) for every element.
- **Eccentricity:** the add-in exports raw positions and rotations. The **engine** computes ex and ey in the footing's local axes. The app can override them through `ecc_override`.

---

## 1. Export file: `footings_export.json`

```json
{
  "schema_version": "0.2.0",
  "exported_at": "2026-10-04T15:52:00-04:00",
  "source": {
    "model_title": "Project_Structure.rvt",
    "model_guid": "....",
    "revit_version": "2026",
    "exported_by": "user.name"
  },
  "units": {
    "length": "in",
    "force": "kip",
    "moment": "kip-in",
    "stress": "ksi",
    "soil_pressure": "ksf"
  },
  "footings": [
    {
      "id": "e3b0c442-98fc-1c14-...-0004a1b2",
      "mark": "F-12",
      "type_name": "Spread Footing - Rect",
      "level": "Foundation",
      "fc": 4.0,
      "geometry": {
        "centroid": [1200.0, 840.0],
        "rotation_deg": 0.0,
        "B": 72.0,
        "L": 72.0,
        "thickness": 24.0,
        "top_elevation": -48.0
      },
      "column": {
        "id": "a1f2c3d4-....-0004c5d6",
        "mark": "C-7",
        "location": [1212.0, 840.0],
        "rotation_deg": 0.0,
        "b": 18.0,
        "h": 18.0,
        "fc": 6.0,
        "load_frame": "global",
        "load_elevation": -48.0,
        "loads": [
          { "case": "D",     "type": "D", "P": 120.0, "Mx": 0.0,  "My": 0.0,  "Vx": 0.0, "Vy": 0.0 },
          { "case": "L",     "type": "L", "P": 60.0,  "Mx": 0.0,  "My": 0.0,  "Vx": 0.0, "Vy": 0.0 },
          { "case": "W-X+",  "type": "W", "P": -10.0, "Mx": 15.0, "My": 30.0, "Vx": 4.0, "Vy": 2.0 },
          { "case": "W-X-",  "type": "W", "P": 10.0,  "Mx": -15.0,"My": -30.0,"Vx": -4.0,"Vy": -2.0 }
        ]
      },
      "input_hash": "sha256:9f86d081..."
    }
  ],
  "skipped": [
    { "id": "....", "mark": "F-31", "reason": "2 columns matched; combined footing, design manually" },
    { "id": "....", "mark": "F-40", "reason": "Column loads missing: no D case" }
  ],
  "warnings": []
}
```

### Field notes

| Field | Notes |
|---|---|
| `column` | A single object, since the scope is one column per footing. A footing with zero or several matched columns goes into `skipped` with a reason. |
| `loads[].type` | One of `D, L, Lr, S, R, W, E`. The combination logic keys on `type`, while `case` is just a label. Several cases of the same type (wind in four directions, seismic in two) are each applied as separate combinations. |
| `load_frame` | `"global"` or `"column_local"`. If the other plugin writes loads in the column's local axes, the engine rotates them using `column.rotation_deg`. This must be confirmed (see Open decisions). |
| `load_elevation` | The elevation where the reported loads act. The engine adds V times (distance to footing base) to the base moment. |
| Unfactored cases only | Do not export combinations. The engine builds them. |
| `fc` (footing and column) | Needed for the bearing check at the column-footing interface. |
| `input_hash` | Hash of everything the design depends on (geometry, fc, column, loads). Echoed back on import to detect stale designs. |
| `skipped` | Footings the add-in refuses to export, with reasons. The app shows these so nothing disappears silently. |

### Not in the export (entered in the app)

Stored as an app-side `project_settings.json`, which can be saved and reloaded per project:

```json
{
  "code": { "concrete": "ACI318-19", "loads": "ASCE7-16" },
  "soil": {
    "qa": 3.0,
    "qa_basis": "net",
    "soil_unit_weight": 0.120,
    "overburden_depth": 36.0,
    "surcharge": 0.1,
    "qa_increase_transient": 1.0
  },
  "limits": { "min_size": 36.0, "max_size": 144.0, "size_increment": 6.0, "min_thickness": 12.0 },
  "reinforcement": { "bar_sizes": ["#5", "#6", "#7"], "cover": 3.0, "fy": 60.0 }
}
```

`qa_increase_transient` defaults to **1.0**. ASCE 7-16 does not permit the allowable stress increase with its ASD combinations. Whether the 1/3 increase is allowed under the alternative basic combinations is a code and geotech decision, so make it explicit and per project.

---

## 2. Load combinations (ASCE 7-16)

Implement in `codes/asce7_16.py`, independent of the ACI module. Each combination is a list of `(type, factor)` terms. Groups like `(Lr or S or R)` expand to one combination per member. Every W case and every E case generates its own set.

**ASD, used for bearing (§2.4.1)**

| # | Combination |
|---|---|
| 1 | D |
| 2 | D + L |
| 3 | D + (Lr or S or R) |
| 4 | D + 0.75L + 0.75(Lr or S or R) |
| 5 | D + (0.6W or 0.7E) |
| 6a | D + 0.75L + 0.75(0.6W) + 0.75(Lr or S or R) |
| 6b | D + 0.75L + 0.75(0.7E) + 0.75S |
| 7 | 0.6D + 0.6W |
| 8 | 0.6D + 0.7E |

**LRFD, used for strength (§2.3.1)**

| # | Combination |
|---|---|
| 1 | 1.4D |
| 2 | 1.2D + 1.6L + 0.5(Lr or S or R) |
| 3 | 1.2D + 1.6(Lr or S or R) + (L or 0.5W) |
| 4 | 1.2D + 1.0W + L + 0.5(Lr or S or R) |
| 5 | 1.2D + 1.0E + L + 0.2S |
| 6 | 0.9D + 1.0W |
| 7 | 0.9D + 1.0E |

Notes (verify against your office standards and the code text):

- In ASCE 7-16, W is already strength level, which is why ASD uses 0.6W. E is strength level too, hence 0.7E.
- **Bearing (ASD):** pressure uses column loads plus footing self-weight and soil on top, compared with `qa` (adjusted for gross vs. net).
- **Strength (LRFD):** design for **net factored upward pressure from the column loads only.** Footing self-weight and overburden cancel against the soil reaction for flexure and shear, so they are excluded.
- Use the 0.6D and 0.9D combinations for uplift and eccentricity checks too, since reduced dead load can govern.
- Check whether the other plugin has already applied live load reductions, so they are not applied twice.

---

## 3. ACI 318-14 vs 318-19: not quite "the same" for footings

Flexure, development, and most detailing are effectively the same. **Shear is where the two editions differ**, and shear often governs footing thickness. My recollection, to be verified against the code text before implementing:

- **One-way shear.** For members without shear reinforcement (all spread footings), 318-19 uses an equation with a longitudinal reinforcement ratio term, ρw, and a size-effect factor λs. 318-14 uses the simple 2λ√f'c·bw·d form. 318-19 can give noticeably lower capacity for lightly reinforced thick footings.
- **Two-way (punching) shear.** 318-19 also applies the size-effect factor λs to vc when there is no shear reinforcement, and λs reduces capacity as d grows.
- **Practical effect:** a footing that passes under 318-14 can fail under 318-19 on thickness alone.

Design consequence: the code module is a **provider with a single switch** (`code_edition`), not two separate implementations. Share everything common in `codes/aci_common.py` and put only the differing equations in `aci318_14.py` and `aci318_19.py`. Hand-calc tests should exist **per edition**, with at least one case where they disagree.

---

## 4. Results file: `footings_results.json`

```json
{
  "schema_version": "0.2.0",
  "engine_version": "0.1.0",
  "generated_at": "2026-10-04T16:30:00-04:00",
  "source_export": "footings_export.json",
  "code": { "concrete": "ACI318-19", "loads": "ASCE7-16" },
  "results": [
    {
      "footing_id": "e3b0c442-....",
      "input_hash": "sha256:9f86d081...",
      "status": "pass",
      "flags": [],
      "governing": "Bearing: D + L",
      "size": { "B": 84.0, "L": 84.0, "thickness": 24.0 },
      "utilization": {
        "bearing": 0.91,
        "flexure": 0.62,
        "one_way_shear": 0.48,
        "punching": 0.77
      },
      "reinforcement": { "bottom_each_way": "#6 @ 12 in" }
    },
    {
      "footing_id": "....",
      "input_hash": "sha256:....",
      "status": "out_of_scope",
      "flags": ["Resultant outside kern under ASD combo 5 (W-X+)"]
    }
  ]
}
```

Only a **small summary** goes back to Revit. The full calc package stays in the app and exports as PDF or markdown.

### Write-back to Revit (shared parameters on the footing)

| Parameter | Type |
|---|---|
| `FTG_Design_B` / `_L` / `_Thickness` | Length |
| `FTG_Design_Status` | Text (pass / fail / out of scope / stale) |
| `FTG_Design_Util` | Number |
| `FTG_Design_Hash` | Text |

Import rules:

1. Look up the element by `UniqueId`. If not found, skip and report.
2. Recompute the hash of the current model state. If it differs from `input_hash`, **skip and mark stale.**
3. Otherwise write the parameters inside one `Transaction`.
4. Show a summary dialog: written / stale / missing / out of scope.

Write suggested sizes to parameters only. Don't resize the footing geometry from the tool.

---

## 5. Python package layout

```text
footing_tool/
├── pyproject.toml
├── src/footing_tool/
│   ├── __init__.py
│   ├── cli.py                   # `footing-tool app --export <path>` and `footing-tool design ...`
│   ├── models/                  # pure data, no logic
│   │   ├── inputs.py            # Footing, Column, LoadCase, SoilSettings, DesignSettings
│   │   ├── results.py           # BearingResult, StrengthResult, FootingResult, Status
│   │   └── schema.py            # load/validate export JSON, dump results JSON
│   ├── core/                    # engine, no Streamlit imports
│   │   ├── geometry.py          # local frame, ex/ey, kern, rotations
│   │   ├── loads.py             # build combos, transfer V to base moment, resultant
│   │   ├── scope.py             # out_of_scope checks, returns flags
│   │   ├── bearing.py           # concentric and within-kern (ASD)
│   │   ├── flexure.py           # Mu at column face, As required, minimum steel
│   │   ├── shear.py             # one-way at d, two-way punching with moment transfer
│   │   ├── interface.py         # bearing at column-footing interface, dowels
│   │   ├── detailing.py         # development length, cover, bar spacing
│   │   ├── sizing.py            # iterate size to qa and round to increments
│   │   └── design.py            # design_footing(...) -> FootingResult
│   ├── codes/
│   │   ├── base.py              # CodeProvider protocol
│   │   ├── asce7_16.py          # ASD and LRFD combinations
│   │   ├── aci_common.py        # phi factors, flexure, development, shared pieces
│   │   ├── aci318_14.py         # edition-specific shear
│   │   └── aci318_19.py         # edition-specific shear (size effect, rho_w)
│   ├── standards/
│   │   └── severud_standard.py  # standard spread footing lookup
│   ├── batch.py                 # run design_footing over a list, collect results
│   ├── internal_forces/         # optional, later
│   │   └── statics.py           # closed-form V, M along sections for diagrams
│   └── report/
│       ├── calc_package.py      # step-by-step calc output (markdown / PDF)
│       └── plots.py             # pressure diagram, footing plan, V/M diagrams
├── app/
│   ├── streamlit_app.py         # reads --export path from CLI args
│   └── pages/
│       ├── 1_Import.py          # show export, skipped list, warnings
│       ├── 2_Settings.py        # qa, soil, code edition, limits
│       ├── 3_One_Off.py         # single footing design and calc view
│       ├── 4_Bulk.py            # table of all footings, run all, save results
│       └── 5_Custom.py          # manual input, no Revit file
└── tests/
    ├── test_geometry.py
    ├── test_combinations.py     # every ASCE 7-16 combo expands correctly
    ├── test_bearing.py
    ├── test_shear_318_14.py
    ├── test_shear_318_19.py     # include a case where editions disagree
    ├── test_flexure.py
    ├── test_scope.py
    ├── test_schema.py
    └── fixtures/sample_export.json
```

### Dependency rules

- `models` imports nothing from the package.
- `core` and `codes` import from `models` (and `core` from `codes`).
- `report` and `batch` import from `core`.
- `app` and `cli` import from everything. Nothing imports from `app`.

### Core API sketch

```python
# models/inputs.py
from dataclasses import dataclass

@dataclass(frozen=True)
class LoadCase:
    case: str
    type: str            # "D" | "L" | "Lr" | "S" | "R" | "W" | "E"
    P: float; Mx: float; My: float; Vx: float; Vy: float

@dataclass(frozen=True)
class Column:
    id: str
    location: tuple[float, float]
    b: float; h: float
    fc: float
    rotation_deg: float
    load_frame: str                  # "global" | "column_local"
    load_elevation: float
    loads: tuple[LoadCase, ...]

@dataclass(frozen=True)
class Footing:
    id: str
    mark: str
    centroid: tuple[float, float]
    B: float; L: float; thickness: float
    top_elevation: float
    fc: float
    rotation_deg: float
    column: Column
    ecc_override: tuple[float, float] | None = None   # (ex, ey), local axes
```

```python
# core/design.py
def design_footing(footing, soil, settings, code) -> FootingResult:
    flags = scope.pre_check(footing, settings)          # geometry/data problems
    if flags:
        return FootingResult.out_of_scope(footing, flags)

    frame = geometry.local_frame(footing)
    asd = loads.at_footing_base(footing, code.asd_combinations(), frame)
    lrfd = loads.at_footing_base(footing, code.lrfd_combinations(), frame)

    flags = scope.post_check(footing, asd, frame)       # kern, uplift
    if flags:
        return FootingResult.out_of_scope(footing, flags)

    bearing_res = bearing.check(footing, asd, soil)
    flex = flexure.check(footing, lrfd, code)           # net pressure, column loads only
    shear_res = shear.check(footing, lrfd, code)        # edition-specific
    return FootingResult.assemble(footing, bearing_res, flex, shear_res)
```

Bulk is `[design_footing(f, ...) for f in footings]`, so there is no second code path. A headless CLI wrapper around the same function lets the add-in or a script run bulk without the UI.

---

## 6. Revit add-in layout (C#)

```text
FootingTool/
├── App.cs                         # ribbon panel
├── Commands/
│   ├── OpenFootingToolCommand.cs  # export, launch app, open browser
│   └── ApplyResultsCommand.cs     # read results.json, hash check, write-back
├── Collectors/
│   ├── FootingCollector.cs
│   ├── ColumnCollector.cs         # includes linked models (with transform)
│   └── ColumnLoadReader.cs        # reads parameters per the map below
├── Config/
│   └── load_parameter_map.json    # maps load type/component to parameter names
├── Matching/
│   └── ColumnFootingMatcher.cs    # exactly one column per footing, else skipped
├── Models/                        # DTOs mirroring the JSON schema
├── Serialization/
│   ├── JsonWriter.cs
│   └── InputHasher.cs             # canonical form, rounded floats, SHA-256
├── Process/
│   └── EngineLauncher.cs          # start, reuse, and stop the local Streamlit process
├── Writeback/
│   └── ResultsApplier.cs
└── UnitConversion.cs
```

### Reading loads from the other plugin's parameters

Do not hard-code parameter names in `ColumnLoadReader`. Put them in `load_parameter_map.json` so a rename in the other plugin is a config edit:

```json
{
  "cases": [
    { "case": "D", "type": "D", "P": "Load_D_P", "Mx": "Load_D_Mx", "My": "Load_D_My", "Vx": "Load_D_Vx", "Vy": "Load_D_Vy" }
  ],
  "load_frame": "global",
  "load_elevation_source": "column_base"
}
```

(Names above are placeholders; use the other plugin's real ones.) On export, the reader should check that required parameters exist and have values, and put any column with gaps into `skipped` with the missing names. Those loads also feed `input_hash`, so a later load change in the model marks the result stale.

### Process and file flow

1. Engineer clicks **Footing Tool** in the ribbon.
2. The add-in exports to a per-model session folder, e.g. `%LOCALAPPDATA%\FootingTool\sessions\<model_guid>\footings_export.json`.
3. `EngineLauncher` runs the local engine, e.g. `python -m footing_tool app --export <path>`, with `--server.headless true` and `--browser.gatherUsageStats false`. It records the process, reuses a running instance, and opens the browser at the local port.
4. The engineer reviews, sets `qa` and so on, designs, and clicks **Save results** in the app, which writes `footings_results.json` into the same session folder.
5. The engineer clicks **Apply Results** in Revit, which reads that file with the hash check.

Use a session folder instead of a file-upload step. It removes a manual upload and avoids the download-then-import dance.

---

## 7. Local deployment

Each engineer needs the add-in and a working Python environment. This is the part most likely to bite later, so decide it early.

- **Python environment:** an installer provisions an isolated environment (venv or embedded Python, ideally built with `uv`) under `%LOCALAPPDATA%\FootingTool\`, pinned to exact versions of the package, `streamlit`, `numpy` and `pandas`. Don't rely on whatever Python is on the machine. Seed it from a wheel cache on a network share so it works without internet.
- **Add-in install:** `.addin` manifest and DLL placed per user (`%APPDATA%\Autodesk\Revit\Addins\<version>\`) so no admin rights are needed.
- **Revit versions and .NET:** Revit 2024 and earlier run on .NET Framework 4.8, Revit 2025 and later on .NET 8. Decide which versions your office supports, and multi-target the add-in if you need both.
- **Versioning:** the add-in and the engine each report a version, and `schema_version` is checked on import. A mismatch should say "update your install", not misbehave.
- **Updates:** a version file on the share that the installer or the add-in checks, so engineers don't run stale engines. Stale engines are a design-quality risk for a structural calc tool.
- **Telemetry and data:** disable Streamlit usage stats, and bind to `localhost` only, since project data stays on the machine.
- **Packaging Streamlit as a single `.exe` (PyInstaller):** possible but fiddly. A managed venv installed by a script is usually less painful for an internal tool.

---

## 8. Milestones and tasks

### M1: Contract (do first)

- [ ] Confirm the other plugin's parameter names, units, load types and frame (see Open decisions)
- [ ] Finalize `footings_export.json` fields and sign convention
- [ ] Write `schema.py`, a JSON Schema file and `sample_export.json`
- [ ] Decide `qa` gross/net handling and transient increase policy
- [ ] Finalize `out_of_scope` triggers and size limits

### M2: Engine MVP (no Revit needed)

- [ ] `asce7_16.py` with ASD and LRFD combinations plus tests
- [ ] `geometry.py`: local frame, ex/ey, kern check
- [ ] `loads.py`: combinations, V-to-base-moment transfer, self-weight and overburden
- [ ] `scope.py`: pre and post checks
- [ ] `bearing.py`: concentric and within kern
- [ ] `sizing.py`: size to qa and round to increments
- [ ] Hand-calc tests for each

### M3: Strength design

- [ ] `aci_common.py` and `flexure.py` (net factored pressure)
- [ ] `shear.py` for 318-14 and 318-19, with a test case where they disagree
- [ ] `interface.py` and `detailing.py`
- [ ] `calc_package.py`: line-by-line output
- [ ] Review the calc package with a senior engineer

### M4: App

- [ ] CLI entry point that accepts `--export <path>`
- [ ] Streamlit pages: import, settings, one-off, bulk, custom
- [ ] Save-results button that writes into the session folder
- [ ] Hook up Severud standard footings table
- [ ] Pressure diagram plot

### M5: Revit add-in

- [ ] Export command: footings, single-column matching, `skipped` list
- [ ] `ColumnLoadReader` with `load_parameter_map.json` and validation
- [ ] Hashing with canonical float rounding
- [ ] `EngineLauncher` (start, reuse, shutdown)
- [ ] Shared parameter set and Apply Results command
- [ ] Summary dialog: written / stale / missing / out of scope
- [ ] Test on a real model, including a linked model, a rotated footing and a rotated column

### M6: Deployment

- [ ] Installer for Python environment plus add-in
- [ ] Network-share wheel cache and version check
- [ ] Decide supported Revit versions and .NET targets
- [ ] Pilot with two or three engineers

### Maybe later (revisit against the scope)

- [ ] V and M diagrams via closed-form statics
- [ ] Partial-contact bearing beyond the kern
- [ ] Tie beams / straps (likely hand-design territory given the "easy footings only" scope)

---

## 9. Open decisions

- **Other plugin's output:** parameter names, units, and which load types it provides (D, L, Lr, S, R, W, E).
- **Load frame:** are the column loads in global axes or the column's local axes?
- **Load location:** at what elevation are the loads reported (column base, top of footing, or somewhere else)?
- **Lateral cases:** are wind and seismic stored as several directional cases, or as one signed set? Is vertical seismic (Ev) already in D?
- **Live load reduction:** already applied by the other plugin?
- **Transient increase:** is the 1/3 `qa` increase permitted on projects, and who decides per project?
- **Supported Revit versions** (affects .NET targets).
- **Standard footing table:** where it lives (CSV, JSON, Python) and who maintains it.
- **Linked models:** support from the start, or require the structural model to be open?
