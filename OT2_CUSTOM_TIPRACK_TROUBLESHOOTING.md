# Troubleshooting: OT‑2 errors with a custom "tip‑rack" labware and tip height

This note diagnoses the failure reported in
[`vertical-cloud-lab/byu-vcl#116`](https://github.com/vertical-cloud-lab/byu-vcl/pull/116)
and [`byu-vcl#33`](https://github.com/vertical-cloud-lab/byu-vcl/issues/33): an OT‑2
cannot reliably pick up / calibrate the wireless color sensor (WCS), which is modeled as
an oversized "tip" using a custom tip‑rack labware definition
(`byu_color_sensor_charging_port.json`, `isTiprack: true`, `tipLength: 84`,
`tipOverlap: 0`).

The investigation here is against the Opentrons robot software (this repository), which is
where the relevant tip‑height math lives.

## TL;DR

* The robot software **ignores the `parameters.tipOverlap` value in a labware
  definition.** Tip overlap is always taken from the **pipette** configuration, and for
  an unknown (custom) tip‑rack it falls back to the pipette's `default` overlap.
* For a `p20_single_gen2`, that default overlap is **8.25 mm**.
* So the custom labware's `tipOverlap: 0` has **no effect**. After "picking up" the WCS,
  the robot models the sensor as `tipLength − 8.25 = 84 − 8.25 = 75.75 mm` long instead of
  the intended 84 mm. Every subsequent move that is referenced to the tip end is therefore
  off by **8.25 mm**, which is what makes calibration/pickup look like it "just misses."
* The fix is to **calibrate tip length for the custom rack on the robot** (primary), and/or
  to **bake the ignored overlap into `tipLength`** in the JSON (no‑calibration workaround):
  set `tipLength` to `84 + 8.25 = 92.25`.

## Why versions matter ("v9 changes")

Older OT‑2 software (≤ 6.x) had a separate "legacy" protocol executor in addition to the
Protocol Engine. Starting with the 7.x line and continuing through 8.x and 9.x, **the
Protocol Engine is the sole executor for every protocol on the OT‑2**, regardless of the
protocol's `apiLevel` (e.g. `2.12`). Tip‑overlap and tip‑length handling are therefore
governed by the Protocol Engine code paths described below.

Crucially, **neither the Protocol Engine nor the legacy helper ever reads
`parameters.tipOverlap` from the labware** — both use the *pipette's* overlap table. So a
custom rack that "worked" on older hardware almost certainly worked because a **tip‑length
calibration had been performed for it**, not because `tipOverlap: 0` was honored. Moving to
a fresh robot / fresh app, or re‑printing the sensor, loses that calibration and re‑exposes
the 8.25 mm error.

## Root cause, with code references

1. The Protocol Engine computes the *nominal effective tip length* as
   `tipLength − nominal_overlap`:

   ```python
   # api/src/opentrons/protocol_engine/state/geometry.py
   def get_nominal_effective_tip_length(self, pipette_id, labware_id) -> float:
       labware_uri = self._labware.get_definition_uri(labware_id)
       nominal_overlap = self._pipettes.get_nominal_tip_overlap(
           pipette_id=pipette_id, labware_uri=labware_uri
       )
       return self._labware.get_tip_length(labware_id=labware_id, overlap=nominal_overlap)
   ```

2. `get_tip_length` subtracts the overlap that was passed in; it **never looks at
   `parameters.tipOverlap`**:

   ```python
   # api/src/opentrons/protocol_engine/state/labware.py
   def get_tip_length(self, labware_id, overlap: float = 0) -> float:
       definition = self.get_definition(labware_id)
       ...
       return definition.parameters.tipLength - overlap
   ```

3. The overlap value comes from the **pipette** config, keyed by tip‑rack URI, and falls
   back to the pipette's `default` for any custom/unknown rack:

   ```python
   # api/src/opentrons/protocol_engine/state/pipettes.py
   def get_nominal_tip_overlap(self, pipette_id, labware_uri: str) -> float:
       tip_overlaps_by_uri = self.get_config(pipette_id).nominal_tip_overlap
       try:
           return tip_overlaps_by_uri[labware_uri]
       except KeyError:
           return tip_overlaps_by_uri.get("default", 0)
   ```

   For `p20_single_gen2` the `default` is `8.25` (see
   `shared-data/pipette/definitions/1/pipetteModelSpecs.json`,
   `p20_single_v2.x → tipOverlap.default`).

4. On pick‑up, the nominal effective length is only used as a *fallback*; a saved
   **tip‑length calibration** for this pipette + rack overrides it:

   ```python
   # api/src/opentrons/protocol_engine/execution/tip_handler.py
   nominal_tip_geometry = self._state_view.geometry.get_nominal_tip_geometry(...)
   actual_tip_length = await self._labware_data_provider.get_calibrated_tip_length(
       pipette_serial=...,
       labware_definition=...,
       nominal_fallback=nominal_tip_geometry.length,  # = tipLength - 8.25 here
   )
   ```

5. The legacy helper (used by older API surfaces) behaves the same way — calibrated value
   if present, otherwise `tipLength − pipette default overlap`:

   ```python
   # api/src/opentrons/protocols/api_support/instrument.py
   try:
       return instr_cal.load_tip_length_for_pipette(...).tip_length
   except TipLengthCalNotFound:
       tip_overlap = pipette["tip_overlap"].get(uri, pipette["tip_overlap"]["default"])
       return tip_rack_definition["parameters"]["tipLength"] - tip_overlap
   ```

### Net effect for the WCS rack

| Situation | Modeled tip length below the nozzle | Error vs. real 84 mm |
| --- | --- | --- |
| No tip‑length calibration | `84 − 8.25 = 75.75 mm` | **8.25 mm too short** |
| Tip‑length calibration done | calibrated (≈ real) | ~0 mm |

An 8.25 mm vertical error is exactly the "this should *definitely* pick it up but it just
misses" behavior seen in the failure video.

## Recommended fixes

### Fix A — Calibrate tip length for the custom rack (primary, do this regardless)

Any custom tip‑rack on an OT‑2 must have a **tip‑length calibration** before its geometry
is trusted. In the Opentrons App: *Robot → Calibration → Tip Length Calibration* (or the
labware/Labware Position Check flow), select the `byu_color_sensor_charging_port` rack and
the left‑mount `p20_single_gen2`, and complete the guided jog. This stores a calibrated
length that overrides the nominal `tipLength − 8.25` fallback (see `tip_handler.py` above),
which removes the 8.25 mm error.

If the *calibration flow itself* errors out, check Fix C (deck/Z envelope) and re‑seat / re‑print
a straight sensor body — a bent or stretched sensor changes the real length and defeats any
calibration.

### Fix B — Compensate for the ignored overlap directly in the JSON (no‑calibration workaround)

Because the engine subtracts the pipette `default` overlap (8.25 mm for `p20_single_gen2`)
and ignores `parameters.tipOverlap`, make the *nominal* model match reality by adding that
overlap back into `tipLength`:

```jsonc
"parameters": {
    "format": "irregular",
    "isTiprack": true,
    "tipLength": 92.25,    // 84 mm real length + 8.25 mm pipette default overlap
    "tipOverlap": 8.25,    // informational only; the engine uses the pipette table
    ...
}
```

With `tipLength = 92.25`, the nominal effective length becomes `92.25 − 8.25 = 84 mm`,
matching the physical sensor even when no tip‑length calibration is present.

> Note: if you change the pipette (e.g. to a GEN1 or a different model), look up that
> pipette's `tipOverlap.default` in `shared-data` and re‑derive `tipLength` accordingly.
> This is why Fix A (calibration) is preferred — it is pipette‑specific and measured.

### Fix C — Watch the Z / deck envelope when carrying an ~84 mm "tip"

An 84 mm "tip" plus a target labware can exceed the OT‑2's usable Z travel and trigger an
out‑of‑bounds / deck‑conflict error that *looks* like a calibration failure. When moving the
sensor over other labware (e.g. the `corning_96_wellplate_360ul_flat` in
`protocol_pick_move_return.py`), keep generous positive `z` offsets on `.top()` moves and
avoid commanding the tip critical point below a well's bottom. Prefer explicit
`plate["A1"].top(z=...)` / `.bottom(z=...)` targets rather than relying on the (overlap‑
adjusted) tip critical point.

## Checklist for the BYU protocols

1. Load the `p20_single_gen2` on the **left** mount (already corrected in PR #116).
2. Perform **tip‑length calibration** for `byu_color_sensor_charging_port` on the robot
   (Fix A).
3. If you cannot calibrate, set the labware `tipLength` to `92.25` (Fix B) so the fallback
   geometry is correct.
4. Re‑verify the two carried‑over offsets after the tip model is corrected — the `−1.3 mm`
   measurement offset (how far below the target well top the sensor is lowered to take a
   reading, used in `protocol_pick_move_return.py`) and the `−80 mm` return‑drop offset (how
   far the sensor is lowered back into its charging port on return). Both were tuned against
   the *previous* (calibrated) geometry and may shift by up to the 8.25 mm error once the
   model is right.
5. Keep generous `z` clearance on transit moves (Fix C).
