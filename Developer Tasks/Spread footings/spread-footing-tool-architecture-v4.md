---
tags: [project, structural, revit, python]
status: design
version: 4
---

# Spread Footing Tool: Design (v4)

## What changed from v3

- **Host model only.** The tool reads, analyzes and resizes only elements in the open (our) model. Nothing is pulled from linked models: no footings, no columns, no loads. We are the engineer of record for our model only; consultants coordinate their own models to match ours.
- **Project file for save / close / reopen.** Each project has a project file holding the engineer's selections, qa overrides and table settings. Reopening it restores the work; DCRs are always recomputed, never restored.
- **Drop-down rule narrowed.** It starts empty on a new project. A selection only ever comes from the engineer, either picked now or saved earlier in the project file. The current Revit size is still never used as a selection.

## What changed from v2

- **The engineer is always the designer.** The tool never selects or suggests a footing. The main screen is a table with a drop-down per footing. DCR appears only after the engineer picks.
- **Standard footings are looked up, not designed.** They are already verified in-house, so the drop-down reads from a library. The full strength-design engine is needed only for the **custom footing designer**.
- **Load types are detected, not required.** Combinations are built from whichever of D, L, Lr, S, R, W, E exist.
- **Removed:** soil unit weight, overburden depth, surcharge, min/max footing size, a bar-size setting, auto-sizing, and the design status / utilization / hash parameters in Revit.
- **Kept:** `qa` basis setting (default **net**), 6 in size increment and 12 in minimum thickness (custom designer only).
- **Code edition and bearing info are app inputs, not Revit data.** Code edition is one table-level setting. `qa` and its basis are a table-level default that any row can override.
- **Write-back resizes the footing** (width, length, thickness). It is the point of the tool, and the safeguards are in section 7.
- **Moments are in kip-ft.**
- **Milestones reordered:** the table with standard footings and DCR ships before the custom designer.

---

## 1. Principles

1. **Revit models, the app designs.** No design logic in C#.
2. **The tool never chooses.** No default selection, no "recommended" tag, no auto-fill of the DCR column.
3. **Nothing is written to Revit unless the engineer picked a footing for that row.**
4. **Anything visible-but-silent is a risk.** If a load type is absent (e.g. no seismic), the app says so on screen instead of quietly dropping those combinations.
5. **The app runs on its own once it has an export file.** Revit is needed only to export data and to apply results.
6. **Our model only.** Only elements owned by the open model are read, analyzed or modified. Linked models are ignored entirely.

---

## 2. Workflow

1. Engineer clicks **Footing Tool** in Revit. The add-in exports footings, columns and loads **from the open model only** (linked models are not collected) to the project folder, then starts the local web app.
2. The app shows one **row per footing**, similar to an Excel table:

| Column | Source | Editable |
|---|---|---|
| Mark / Level | Revit | no |
| f'c (footing) | Revit | no |
| Column size, f'c | Revit | no |
| qa, qa basis | table default, or this row's override | yes (override per row) |
| Load types used | detected from the loads | no |
| ex, ey | computed from positions | override allowed |
| Governing ASD / LRFD demand | computed | no |
| Current footing size | Revit | no |
| **Selected footing** | **drop-down** | **yes** |
| **DCR** | computed after selection | no |
| Flags | e.g. outside kern, uplift | no |

3. The drop-down lists **standard footings matching the table's code edition, the row's f'c and the row's effective qa**, plus the user's **custom footings** with the same basis. If there is no match, the row says so and the engineer can open the custom footing designer. **The drop-down starts empty** on a new project. When a saved project file is opened, the engineer's own saved selections are restored (see *Project file*). The footing's current Revit size is never used to fill the drop-down; it is shown only in its own read-only column.

**Table-level settings** sit in a bar above the table:

- **Code edition** (ACI 318-14 or 318-19).
- **Default qa and qa basis** (basis defaults to net). Every row uses these unless it has its own value.
- **Per-row override:** the qa and basis cells in each row are editable. An overridden cell is visually highlighted and has a "reset to default" control. Changing the default later updates only the rows that are not overridden, so a deliberate override is never silently replaced.
- **Persistence:** these settings are saved in the project file (see below).
- **If a setting change invalidates a selection** (for example a new code edition or qa means the chosen footing no longer matches the filter), the row shows "selection no longer valid" with the DCR blank. It does not keep the old DCR or switch to another footing.
4. DCR recalculates every time a selection changes.
5. **Apply to Revit** lists old size → new size for every row with a selection, asks for confirmation, then resizes those footings.

### Project file (save, close, reopen)

Each project has one project file, `<project>.footingproj.json`, in the project folder. It stores the engineer's work so it can be saved, closed and reopened later:

- the export file it is tied to
- table settings: code edition, default qa and basis, gross overburden
- per-row qa / basis overrides
- per-row selected footing (standard library ID or custom footing ID)
- the standard library version in use when saved

It does **not** store DCRs. On reopen:

1. Selections and overrides are restored by footing ID.
2. DCRs are recomputed from the current export, library and code edition.
3. A restored selection that no longer matches the filter (different f'c, qa, code edition, or a library entry that was withdrawn) shows **"selection no longer valid"** with the DCR blank. It is never swapped for another footing.
4. If the export has been refreshed, a row whose snapshot hash changed shows **"changed since saved"** and its selection must be re-confirmed by the engineer before it can be applied. Footings no longer in the export are listed as removed; new footings appear with an empty drop-down.
5. If the library version differs from the one saved, a banner says so.

**Save** and **Save as** are on the toolbar. The app warns on close with unsaved changes.

### Running without Revit

The export file contains everything the app needs: footings, columns, loads and the snapshot hashes. So:

- The app can be started on its own (`footing-tool app`) and pointed at a project file with **Open project**, or at an export file with **Open export file** to start a new project. Launching from the Revit button just does this automatically.
- The engineer can select footings, check DCRs, use the custom designer, and save the results file with Revit closed.
- **Revit is required only for two steps:** exporting (Revit model open) and applying results (Revit model open). Apply runs the snapshot check against the live model, so a results file made from an old export is caught instead of applied blindly.

---

## 3. Data exchange

### Conventions

- **Units in the exchange files:** lengths in **inches**, forces in **kip**, **moments in kip-ft**, concrete strength in **ksi**, soil pressure in **ksf**. The units block is written into the file. Convert in exactly one place on each side, because mixing inches and feet is the classic way to get a wrong answer.
- **Coordinates:** global project coordinates.
- **Sign convention:** P positive downward (compression). Mx and My follow the right-hand rule about global X and Y.
- **Keys:** `Element.UniqueId` strings.

### `footings_export.json`

```json
{
  "schema_version": "0.3.0",
  "exported_at": "2026-10-13T10:40:00-04:00",
  "source": { "model_title": "Project_Structure.rvt", "model_guid": "....", "revit_version": "2026", "exported_by": "user.name" },
  "units": { "length": "in", "force": "kip", "moment": "kip-ft", "stress": "ksi", "soil_pressure": "ksf" },
  "footings": [
    {
      "id": "e3b0c442-....",
      "mark": "F-12",
      "type_name": "Spread Footing - Rect",
      "level": "Foundation",
      "fc": 4.0,
      "geometry": {
        "centroid": [1200.0, 840.0],
        "rotation_deg": 0.0,
        "B": 72.0, "L": 72.0, "thickness": 24.0,
        "top_elevation": -48.0
      },
      "column": {
        "id": "a1f2c3d4-....",
        "mark": "C-7",
        "location": [1212.0, 840.0],
        "rotation_deg": 0.0,
        "b": 18.0, "h": 18.0, "fc": 6.0,
        "load_frame": "global",
        "load_elevation": -48.0,
        "loads": [
          { "case": "D",    "type": "D", "P": 120.0, "Mx": 0.0,  "My": 0.0,  "Vx": 0.0, "Vy": 0.0 },
          { "case": "L",    "type": "L", "P": 60.0,  "Mx": 0.0,  "My": 0.0,  "Vx": 0.0, "Vy": 0.0 },
          { "case": "W-X+", "type": "W", "P": -10.0, "Mx": 15.0, "My": 30.0, "Vx": 4.0, "Vy": 2.0 }
        ]
      },
      "snapshot_hash": "sha256:9f86d081..."
    }
  ],
  "skipped": [
    { "id": "....", "mark": "F-31", "reason": "2 columns matched; design manually" },
    { "id": "....", "mark": "F-40", "reason": "No dead load (D) found on column" }
  ]
}
```

Notes:

- **Load type detection:** a type counts as present if its parameters exist and are non-blank. A value of zero still counts as present. If the export has no `E` loads, no seismic combinations are generated. D is the only required type.
- **`snapshot_hash`** is a hash of everything the design depends on (footing geometry, f'c, column position and size, loads). It lives in the export file, not in Revit, since you have no spare parameters. It is how the add-in later detects that the model changed after export.
- **Code edition, `qa` and `qa_basis` are not in the export.** They are entered in the app (table-level default plus per-row override) and saved in the project file. Changing them is not a model change, so they are not part of the snapshot hash, and they never write back to Revit.
- **Host model only:** every footing, column and load in this file comes from the open model. Elements in linked models are never collected. A footing in our model whose column exists only in a linked model has no matching column and goes into `skipped` with the reason "no column in this model".
- The load standard is fixed at ASCE 7-16 for now.
- **Eccentricity:** the file carries raw positions and rotations, and the app computes ex and ey. If eccentricity already exists as a parameter in Revit, tell me and I'll export it as well and cross-check it against the computed value.

### `footings_results.json` (written by the app)

Only rows with a selection are included:

```json
{
  "schema_version": "0.3.0",
  "source_export": "footings_export.json",
  "code_edition": "ACI318-19",
  "load_types_considered": ["D", "L", "W"],
  "results": [
    {
      "footing_id": "e3b0c442-....",
      "snapshot_hash": "sha256:9f86d081...",
      "bearing_used": { "qa": 3.0, "qa_basis": "net", "overridden": false },
      "selected": { "source": "standard", "ref": "SF-72-4-3", "B": 84.0, "L": 84.0, "thickness": 18.0 },
      "dcr": 0.91
    }
  ]
}
```

The DCR is kept in this file for the record but is not written to Revit.

---

## 4. Load handling (ASCE 7-16)

You'll get the **base load cases** (D, L, and so on) and the tool builds both sets. The current plugin writing only an LRFD demand is not enough, since ASD bearing can't be reliably recovered from combined LRFD numbers.

**Detection rule:** build every combination below that involves only load types present. Combinations that reference a missing type are dropped. Groups like `(Lr or S or R)` expand into one combination per type present. Each W case and each E case generates its own combinations.

The app always prints **"Load types considered: D, L, W"** above the table and in the results file. When seismic comes into the project later, nothing in the tool needs to change; the extra combinations appear once those loads exist.

**ASD (§2.4.1), used for bearing**

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

**LRFD (§2.3.1), used for strength**

| # | Combination |
|---|---|
| 1 | 1.4D |
| 2 | 1.2D + 1.6L + 0.5(Lr or S or R) |
| 3 | 1.2D + 1.6(Lr or S or R) + (L or 0.5W) |
| 4 | 1.2D + 1.0W + L + 0.5(Lr or S or R) |
| 5 | 1.2D + 1.0E + L + 0.2S |
| 6 | 0.9D + 1.0W |
| 7 | 0.9D + 1.0E |

(Verify against the code text and office standards.)

Processing steps for each footing:

1. Rotate loads into global axes if `load_frame` is `column_local`.
2. Transfer shear to the footing base: add V × (distance from load elevation to footing base) to the moment.
3. Combine per the lists above.
4. Compute ex and ey in the footing's local axes.
5. Within the kern: peak pressure q_max = P/A × (1 + 6·ex/B + 6·ey/L).
6. Flag outside-kern resultants and net uplift. These are **flags on the row, not auto-designs.** The engineer sees them and decides.

**qa basis:**

- **Net (default):** bearing demand is column load only. Footing self-weight and soil above are already in the net value.
- **Gross:** needs self-weight and soil above. You said no soil parameters, so when the engineer switches to gross, ask for **one number**: the overburden pressure in ksf. Footing self-weight comes from the selected footing at 150 pcf. This avoids reintroducing the unit weight, depth and surcharge inputs, and avoids silently under-estimating the demand.

`qa_increase_transient` stays in settings, default 1.0.

---

## 5. Standard footing library and DCR

### Library entry (proposed shape)

```json
{
  "library_version": "2026-10-01",
  "footings": [
    {
      "id": "SF-72-4-3",
      "name": "6'-0\" sq x 18\" thick",
      "code_edition": "ACI318-19",
      "fc": 4.0,
      "qa": 3.0,
      "B": 72.0, "L": 72.0, "thickness": 18.0,
      "reinforcement": "(7) #6 EW BOT",
      "capacity": { "P_service_max": 108.0, "P_factored_max": 151.0 },
      "verified_by": "initials",
      "verified_on": "2026-09-15",
      "notes": ""
    }
  ]
}
```

- **Filter keys:** the table's code edition, the row's f'c, and the row's effective qa (override if set, otherwise the table default). Match qa with a small tolerance on floats. The drop-down shows only matches.
- **Custom footings** carry the same fields, saved per user in a local JSON, and appear in the same drop-down. They are filtered by the same keys because each was designed for a specific code, f'c and qa.
- The library should be **versioned** and carry who verified it and when. The results file records the library version.

### DCR (proposed)

Standard footings are verified for a maximum load, so DCR is a ratio, not a redesign:

- **Bearing DCR:** governing ASD demand ÷ `P_service_max`. For an eccentric load within the kern, use the effective concentric load that gives the same peak pressure: P_eff = P × (1 + 6·ex/B + 6·ey/L). That makes it equal to q_max / qa.
- **Strength DCR (if the library carries it):** governing LRFD demand ÷ `P_factored_max`, with the same eccentricity treatment.
- **Row DCR:** the larger of the two. Display both on hover.

The **demand side is still a calculation** (combinations, base-moment transfer, eccentricity), but there is no design calculation for a standard footing.

**This depends on what your in-house verification actually records.** If it only records one service load per footing, there is no strength DCR and the LRFD side exists only for custom footings. If it assumes concentric loads only, we need to decide whether the P_eff approach above is acceptable for eccentric footings. See Open decisions.

---

## 6. Custom footing designer

A separate page where the engineer defines a footing (B, L, thickness, **bottom reinforcement each way only**, any bar size), sees the full checks, and saves it. Saved footings then appear in the drop-down.

- **Size increment** 6 in and **minimum thickness** 12 in remain as defaults here (editable).
- **Bar size** is a per-footing choice in this page, not a setting. Any size is allowed.
- **No min/max footing size.**
- **Checks:** bearing (ASD), flexure at column face, one-way shear at d, two-way punching shear with moment transfer, minimum steel, development length, bearing at the column-footing interface.
- **Strength demand** is net factored upward pressure from the column loads only. Footing self-weight and overburden cancel against the soil reaction for flexure and shear.
- **Output:** a line-by-line calc the engineer can read and print, plus the utilization for each check. The custom footing's DCR is the largest of those.

### ACI 318-14 vs 318-19

Flexure, development and most detailing are effectively the same. **Shear differs**, and it often governs footing thickness. From memory, to be verified against the code text before implementing:

- **One-way shear** (no shear reinforcement): 318-19 uses a form with a longitudinal reinforcement ratio term and a size-effect factor λs. 318-14 uses 2λ√f'c·bw·d.
- **Two-way shear:** 318-19 also applies the size-effect factor λs to vc with no shear reinforcement.
- **Effect:** a footing that passes under 318-14 can fail under 318-19 on thickness alone.

Implement as one provider with an edition switch, sharing common code and holding only the differing shear equations per edition. Tests per edition, with at least one case where they disagree.

---

## 7. Write-back to Revit: resizing footings

Only **footing size** is written (width, length, thickness). There are no status, DCR or hash parameters.

### What happens when the engineer clicks Apply (plain language)

1. **Nothing is touched without a selection.** Only rows where the engineer picked a footing are considered. Every other footing in the model is left alone.
2. **A preview comes first.** A dialog lists each footing as "F-12: 72 × 72 × 24 → 84 × 84 × 18" and asks the engineer to confirm.
3. **For each footing, the tool finds it in the model** using the ID saved at export. If the footing has been deleted since, the tool skips it and lists it.
4. **It checks that the footing hasn't changed since export.** It compares the footing's current size and position, its column, and the column loads against the snapshot taken at export. If anything is different, for example someone moved the column or the loads were updated, the tool does **not** resize that footing and lists it as "changed since export, re-export to continue". This stops a design based on old information from being applied to a newer model.
5. **If the footing is unchanged, it resizes** to the selected size. All resizes happen as one Revit action, so a single Undo reverses the whole batch.
6. **If the footing can't be changed**, for example another user has it checked out in a workshared model, or the family doesn't expose the size parameters, the tool skips it and says why.
7. **A summary dialog** lists footings updated, skipped because deleted, skipped because changed, and skipped because locked or unsupported.

Apply only ever touches elements in the open model; footings in linked models never enter the tool. Because the snapshot lives in the export file, this check needs no extra Revit parameters.

### Revit details to settle

- **Type vs instance parameters.** If B, L and thickness are *type* parameters, changing them would resize every footing of that type. The add-in must either write instance parameters, or duplicate / switch to a type with the right size. Which one depends on how the footing family is built.
- **Anchor point.** Default: resize **about the footing centroid**, so ex and ey don't change. If an engineer later wants to hold an edge (property line), that changes eccentricity, so the DCR has to be recomputed for the new geometry. Keep as a later option.
- **Elevation.** When thickness changes, hold the **top of footing** fixed so the bottom drops or rises. Confirm this matches how your families are set up.
- **Rotated footings.** Apply width and length in the footing's local axes.

---

## 8. Python package layout

```text
footing_tool/
├── pyproject.toml
├── src/footing_tool/
│   ├── cli.py                    # `footing-tool app --export <path>`
│   ├── models/
│   │   ├── inputs.py             # Footing, Column, LoadCase, Settings
│   │   ├── library.py            # StandardFooting, CustomFooting, Capacity
│   │   ├── results.py            # Demand, Dcr, CustomFootingResult, Flags
│   │   └── schema.py             # export/results/library validation
│   ├── core/
│   │   ├── geometry.py           # local frame, ex/ey, kern
│   │   ├── loads.py              # detect types, build combos, base moment, resultant
│   │   ├── demand.py             # governing ASD / LRFD demand per footing, flags
│   │   ├── dcr.py                # DCR for a standard footing from library capacity
│   │   └── custom/               # used only by the custom footing designer
│   │       ├── bearing.py
│   │       ├── flexure.py
│   │       ├── shear.py
│   │       ├── interface.py
│   │       ├── detailing.py
│   │       └── design.py         # design_custom_footing(...) -> CustomFootingResult
│   ├── codes/
│   │   ├── base.py
│   │   ├── asce7_16.py           # combinations (types detected)
│   │   ├── aci_common.py
│   │   ├── aci318_14.py
│   │   └── aci318_19.py
│   ├── library/
│   │   ├── standard.py           # load + filter by code, fc, qa
│   │   ├── custom_store.py       # per-user custom footings (local JSON)
│   │   └── data/standard_footings.json
│   └── report/
│       ├── calc_package.py
│       └── plots.py
├── app/
│   ├── streamlit_app.py          # reads --export path
│   └── pages/
│       ├── 1_Footing_Table.py    # main screen: table, drop-downs, DCR, Apply
│       ├── 2_Custom_Footing.py   # custom designer
│       └── 3_Settings.py         # code edition, qa basis, increments
└── tests/
    ├── test_load_detection.py    # only present types produce combos
    ├── test_combinations.py
    ├── test_geometry.py
    ├── test_demand.py
    ├── test_library_filter.py
    ├── test_dcr.py
    ├── test_custom_bearing.py
    ├── test_custom_shear_318_14.py
    ├── test_custom_shear_318_19.py
    └── fixtures/{sample_export.json, sample_library.json}
```

Rules: `models` imports nothing; `core` and `codes` import `models` (and `core` imports `codes`); `app` imports everything; nothing imports `app`. A common `Dcr` result type is shared by standard and custom footings, so the table doesn't care where a DCR came from.

---

## 9. Revit add-in layout (C#)

```text
FootingTool/
├── App.cs
├── Commands/
│   ├── OpenFootingToolCommand.cs  # export, launch app
│   └── ApplyResultsCommand.cs     # preview, snapshot check, resize, summary
├── Collectors/
│   ├── FootingCollector.cs        # open model only; linked models excluded
│   ├── ColumnCollector.cs         # open model only; linked models excluded
│   └── ColumnLoadReader.cs        # reads params via the map; detects load types present
├── Config/load_parameter_map.json # load type/component -> parameter names
├── Matching/ColumnFootingMatcher.cs   # exactly one column per footing, else skipped
├── Models/
├── Serialization/{JsonWriter.cs, SnapshotHasher.cs}
├── Process/EngineLauncher.cs
├── Writeback/FootingResizer.cs    # instance-parameter or type handling, anchor, top elevation
└── UnitConversion.cs
```

`load_parameter_map.json` lists the known load types and component parameter names. The reader exports whichever are present for each column and records which types it found. Missing D puts the footing in `skipped`.

---

## 10. Local deployment

- **Python environment:** an installer sets up an isolated environment (venv or embedded Python, built with `uv`) under `%LOCALAPPDATA%\FootingTool\`, pinned to exact versions of the package, `streamlit`, `numpy`, `pandas`. Seed from a wheel cache on a network share.
- **Add-in:** `.addin` manifest and DLL installed per user, no admin rights.
- **.NET:** Revit 2024 and earlier use .NET Framework 4.8, Revit 2025 and later use .NET 8. Multi-target if you support both.
- **Versioning:** add-in, engine, schema and **standard footing library** each carry a version. A mismatch gives a clear "update your install" message.
- **Library distribution:** the standard footing library is the most safety-critical data file. Keep it in one authoritative place (e.g. the share), read-only for engineers, with the version shown in the app and recorded in the results.
- **Local only:** bind to `localhost`, disable Streamlit usage stats.
- **Custom footings** live per user in `%LOCALAPPDATA%\FootingTool\custom_footings.json`. Decide whether they should be shareable.

---

## 11. Milestones

### M1: Contract

- [ ] Get base load cases (D, L, ...) exported by the load plugin, not just LRFD demand
- [ ] Confirm what the standard footing verification records (see Open decisions) and fix the library format
- [ ] Confirm whether eccentricity already exists as a parameter in Revit
- [ ] Finalize export schema, sign convention, kip-ft handling
- [ ] Write `schema.py`, JSON Schema and `sample_export.json`

### M2: Demand engine (no Revit needed)

- [ ] `asce7_16.py` with type detection and tests
- [ ] `geometry.py`: local frame, ex/ey, kern
- [ ] `loads.py`: rotation, base-moment transfer, combos
- [ ] `demand.py`: governing ASD and LRFD, flags (outside kern, uplift)
- [ ] Hand-calc tests

### M3: Table app MVP (standard footings only)

- [ ] `library/standard.py` with filtering by code, f'c, qa
- [ ] `dcr.py` and tests
- [ ] Streamlit footing table with drop-down and live DCR
- [ ] "Load types considered" banner, flags column
- [ ] Table settings bar: code edition, default qa and basis (plus gross overburden input), per-row qa override, saved per model
- [ ] "Selection no longer valid" handling when settings change
- [ ] Save results file
- [ ] Open export file control, so the app runs without Revit
- [ ] Project file: save, save as, open, unsaved-changes warning
- [ ] On reopen: restore selections and overrides, recompute DCRs, flag invalid / changed / removed rows (tests for each)
- [ ] Drop-down empty on a new project, never filled from current Revit size (test for it)

### M4: Revit add-in

- [ ] Export: footings, single-column matching, load detection, `skipped`; open model only, with a test that elements in linked models are not collected
- [ ] Snapshot hashing with canonical rounding
- [ ] Engine launcher
- [ ] Resizer: type vs instance, centroid anchor, top elevation, rotation
- [ ] Apply flow: preview, snapshot check, one transaction, summary
- [ ] Test on a real model (with consultant links loaded, to confirm nothing from them is collected; rotated footing, rotated column, workshared)

### M5: Custom footing designer

- [ ] `custom/` checks: bearing, flexure, shear (both editions), interface, detailing
- [ ] Bottom-each-way reinforcement input, any bar size
- [ ] Calc package output
- [ ] Custom footing store and drop-down integration
- [ ] Senior engineer review

### M6: Deployment

- [ ] Installer, wheel cache, version checks
- [ ] Library distribution and read-only policy
- [ ] Pilot with 2–3 engineers

### Later

- [ ] Hold-an-edge resizing (recomputes eccentricity and DCR)
- [ ] Pressure and V/M diagrams
- [ ] Shareable custom footings

---

## 12. Open decisions

- **Standard footing verification:** does it record one service load per footing, a factored load, or both? Does it assume concentric loading? This sets the DCR definition (section 5).
- **Eccentric footings:** is the P_eff approach (equivalent concentric load for the same peak pressure) acceptable to use against the table, or should eccentric footings be flagged as needing a custom check?
- **Revit footing families:** are width, length and thickness instance or type parameters? Is thickness measured from a fixed top of footing?
- **Load plugin:** parameter names, units, load frame, load elevation, and whether wind and seismic are multiple directional cases. Also whether live load reduction is already applied.
- **Transient `qa` increase** policy (default 1.0).
- **Supported Revit versions.**
- **Standard library owner** and update process.
- **Project file location:** next to the Revit model on the project share, or in a fixed FootingTool folder per project?
