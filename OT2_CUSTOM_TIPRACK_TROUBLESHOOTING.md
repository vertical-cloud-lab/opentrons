# Troubleshooting: OT‑2 errors with a custom "tip‑rack" labware and tip height

This note diagnoses the failure reported in
[`vertical-cloud-lab/byu-vcl#116`](https://github.com/vertical-cloud-lab/byu-vcl/pull/116)
and [`byu-vcl#33`](https://github.com/vertical-cloud-lab/byu-vcl/issues/33): an OT‑2
throws an **actual error while calibrating** the wireless color sensor (WCS), which is
modeled as an oversized "tip" using a custom tip‑rack labware definition
(`byu_color_sensor_charging_port.json`, `isTiprack: true`, `tipLength: 84`,
`tipOverlap: 0`).

The investigation here is against the Opentrons robot software (this repository), where the
tip‑height math lives, and against the **canonical, hardware‑validated reference** the BYU
work is based on:
[`AccelerationConsortium/ac-dev-lab`](https://github.com/AccelerationConsortium/ac-dev-lab)
(`src/ac_training_lab/ot-2/_scripts/`), plus the build guide at
[`AccelerationConsortium/wireless-color-sensor`](https://github.com/AccelerationConsortium/wireless-color-sensor).

## TL;DR

* The error is thrown **in the Opentrons App's calibration flow** (tip‑length calibration /
  Labware Position Check), not at protocol analysis time. The BYU protocols analyze/simulate
  cleanly; the failure is on the robot during the App's interactive calibration of this
  oversized custom "tip rack."
* **The AC reference never calibrates this rack through the App.** Its `device.py` runs the
  protocol *directly on the robot* (via Jupyter/SSH/Prefect, using
  `opentrons.simulate.get_protocol_api`), calls `pick_up_tip()` on the custom rack
  programmatically, and reaches the measurement/charging positions with **hard‑coded `z`
  offsets** — so the App calibration flow that throws the error is bypassed entirely. This is
  the single most important difference from the BYU setup and the recommended fix.
* The robot software also **ignores the `parameters.tipOverlap` value in a labware
  definition.** Tip overlap is always taken from the **pipette** configuration, falling back
  to the pipette's `default` for an unknown custom rack. For a `p20_single_gen2` that default
  is **8.25 mm**, so the BYU `tipOverlap: 0` has no effect (and the AC reference simply omits
  the field). Absent a saved tip‑length calibration the sensor is modeled
  `84 − 8.25 = 75.75 mm` long, which also shifts every tip‑referenced move by 8.25 mm.
* Recommended fixes, in order: **(A)** run the protocol the way the AC reference does —
  programmatic `pick_up_tip` + hard‑coded offsets, no App calibration (on‑robot, or
  headlessly over the robot's **HTTP API** — see Fix A‑HTTP); **(B)** if you must use
  the App, perform tip‑length calibration for the rack on the robot; **(C)** as a
  no‑calibration geometry workaround, bake the ignored overlap into `tipLength`
  (`84 + 8.25 = 92.25`).

## The canonical AC reference (what actually works on hardware)

The BYU labware and protocols are rebrands of the AC reference. The reference's on‑robot
driver is
[`src/ac_training_lab/ot-2/_scripts/prefect/device.py`](https://github.com/AccelerationConsortium/ac-dev-lab/blob/main/src/ac_training_lab/ot-2/_scripts/prefect/device.py):

```python
import opentrons.simulate
protocol = opentrons.simulate.get_protocol_api("2.12")   # run ON the robot, not via the App
protocol.home()

with open("../ac_color_sensor_charging_port.json") as f1:
    tiprack_2 = protocol.load_labware_from_definition(json.load(f1), 10)
# ...
p300 = protocol.load_instrument("p300_single_gen2", mount="right", tip_racks=[tiprack_1])

@flow
def move_sensor_to_measurement_position(mix_well):
    p300.pick_up_tip(tiprack_2["A2"])            # programmatic pickup, no App calibration
    p300.move_to(plate[mix_well].top(z=-1.3))    # hard‑coded measurement offset

@flow
def move_sensor_back():
    p300.drop_tip(tiprack_2["A2"].top(z=-80))    # hard‑coded return‑drop offset
```

Key properties of the reference, and how the BYU setup diverged:

| Aspect | AC reference (`ac-dev-lab`) | BYU PR #116 |
| --- | --- | --- |
| Execution path | On‑robot `opentrons.simulate.get_protocol_api` served via Prefect/MQTT (Jupyter/SSH) — **App calibration never used** | Uploaded to the **Opentrons App**, then ran the App's tip/labware **calibration** → error |
| Pipette / mount | `p300_single_gen2`, **right** mount | `p20_single_gen2`, **left** mount |
| Custom labware | `ac_color_sensor_charging_port.json`, `tipLength: 84`, **no `tipOverlap`** | `byu_color_sensor_charging_port.json`, `tipLength: 84`, `tipOverlap: 0` (ignored anyway) |
| Tip pickup | `pick_up_tip()` in code, nominal geometry | App calibration flow |
| Reaching targets | Hard‑coded `z` offsets (`-1.3` measure, `-80` return) | Same offsets, but only after App calibration |

Because the reference drives `pick_up_tip` programmatically and never asks the App to
calibrate an 84 mm "tip," it sidesteps the calibration error the BYU team hit. The reference
labware also **omits `tipOverlap` entirely** — confirming the field is not what makes this
work (the robot ignores it; see below).

## Why versions matter ("v9 changes")

Older OT‑2 software (≤ 6.x) had a separate "legacy" protocol executor in addition to the
Protocol Engine. Starting with the 7.x line and continuing through 8.x and 9.x, **the
Protocol Engine is the sole executor for every protocol on the OT‑2**, regardless of the
protocol's `apiLevel` (e.g. `2.12`). Tip‑overlap and tip‑length handling are therefore
governed by the Protocol Engine code paths described below.

Crucially, **neither the Protocol Engine nor the legacy helper ever reads
`parameters.tipOverlap` from the labware** — both use the *pipette's* overlap table. So a
custom rack "works" not because `tipOverlap: 0` (or any labware overlap) is honored, but
because the AC reference avoids the App calibration that throws and drives `pick_up_tip`
programmatically (and, where needed, a saved tip‑length calibration overrides the nominal
geometry). Moving to the App's calibration flow — or to a fresh robot / fresh app, or
re‑printing the sensor — re‑exposes both the thrown calibration error and the 8.25 mm
geometry shift.

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

An 8.25 mm vertical error compounds the primary problem: even after you get past the App
calibration error, a mis‑modeled tip length shifts every tip‑referenced move, consistent with
the "this should *definitely* pick it up but it just misses" behavior seen in the failure
video.

## Recommended fixes

### Fix A — Run it like the AC reference: programmatic pickup, no App calibration (primary)

The reference avoids the calibration step that throws by executing the protocol **directly on
the robot** instead of going through the Opentrons App's tip/labware calibration flow:

* Put the protocol on the robot and run it with `opentrons_execute` over SSH, from the
  robot's Jupyter notebook, or via the Prefect/MQTT `serve` pattern in `device.py` —
  i.e. `opentrons.simulate.get_protocol_api("2.12")` executed on the OT‑2.
* Call `pipette.pick_up_tip(charging_port["A2"])` in code; do **not** run the App's tip‑length
  calibration or Labware Position Check for the WCS rack.
* Reach the measurement / charging positions with explicit hard‑coded offsets
  (`plate[well].top(z=-1.3)`, `charging_port["A2"].top(z=-80)`), exactly as the reference does.

This is the configuration that is validated on hardware. The App calibration flow is the
thing that errors for an oversized custom "tip rack," so the reliable fix is to not use it.

### Fix A‑HTTP — Drive it over the robot's HTTP API instead of the App

If you want to stay off the robot's shell (no SSH / Jupyter) but still bypass the App's
calibration flow, talk to the same robot software directly over its **HTTP API**
(`robot-server`, port `31950`). This is the headless equivalent of Fix A: you enqueue the
exact same Protocol‑Engine commands (`loadPipette`, `loadLabware`, `pickUpTip`, `moveToWell`,
`dropTip`) that `pick_up_tip()` / `move_to()` produce on‑robot, so the App's tip‑length /
Labware Position Check step that throws is never invoked. The `POST /runs/{runId}/commands`
endpoint documents this explicitly: *"You can create a protocol purely over HTTP using
protocol commands"* (`robot-server/robot_server/runs/router/commands_router.py`).

Every request needs the version header `Opentrons-Version` (use `*` for the latest, or a
number ≥ `2`; see `robot-server/robot_server/versioning.py`, `API_VERSION_HEADER`). Replace
`<ROBOT_IP>` with the OT‑2's address.

```bash
BASE=http://<ROBOT_IP>:31950
# Use explicit -H flags so the version header (note the literal "*") is sent verbatim.
post_cmd() {
  curl -s -H 'Opentrons-Version: *' -H 'Content-Type: application/json' \
    -X POST "$BASE/runs/$RUN/commands?waitUntilComplete=true" -d "$1"
}

# 1. Create an empty run (no protocol file).
RUN=$(curl -s -H 'Opentrons-Version: *' -H 'Content-Type: application/json' \
  -X POST $BASE/runs -d '{"data":{}}' | jq -r .data.id)

# 2. Register the custom "tip rack" definition with this run (body must be {"data": <def>}).
curl -s -H 'Opentrons-Version: *' -H 'Content-Type: application/json' \
  -X POST $BASE/runs/$RUN/labware_definitions \
  -d "{\"data\":$(cat byu_color_sensor_charging_port.json)}"

# 3. Enqueue setup commands. With data.source == "setup" and ?waitUntilComplete=true
#    they execute immediately, in order — no `play` action required. Capture the ids the
#    load* commands return and reuse them as pipetteId / labwareId below.
post_cmd '{"data":{"commandType":"loadPipette","params":{
  "pipetteName":"p20_single_gen2","mount":"left"}}}'                     # -> <pipetteId>
post_cmd '{"data":{"commandType":"loadLabware","params":{
  "location":{"slotName":"10"},"loadName":"byu_color_sensor_charging_port",
  "namespace":"custom_beta","version":1}}}'                             # -> <tiprackId>
post_cmd '{"data":{"commandType":"loadLabware","params":{
  "location":{"slotName":"1"},"loadName":"corning_96_wellplate_360ul_flat",
  "namespace":"opentrons","version":1}}}'                               # -> <plateId>
post_cmd '{"data":{"commandType":"pickUpTip","params":{
  "pipetteId":"<pipetteId>","labwareId":"<tiprackId>","wellName":"A2"}}}'
post_cmd '{"data":{"commandType":"moveToWell","params":{
  "pipetteId":"<pipetteId>","labwareId":"<plateId>","wellName":"A1",
  "wellLocation":{"origin":"top","offset":{"x":0,"y":0,"z":-1.3}}}}}'    # hard‑coded offset
```

Notes:

* Commands tagged `"setup"` (the default for ad‑hoc commands) run as soon as they are
  enqueued, so for an interactive driver you do not need `POST /runs/{runId}/actions`
  (`{"data":{"actionType":"play"}}`) at all. Use the `play` action only if you switch to
  `intent: "protocol"` commands.
* Capture the `data.id` returned by each `loadPipette` / `loadLabware` response and feed it
  back as `pipetteId` / `labwareId` — the engine references loaded items by id, not by name.
* This path uses the **same** nominal tip geometry as Fix A, so the 8.25 mm overlap shift
  still applies unless you either save a tip‑length calibration or apply the Fix C
  `tipLength` compensation below.

The Opentrons HTTP API can be driven from any HTTP client — `curl`, Python `requests`, or
the Opentrons HTTP API JS client — all against these same `robot-server` endpoints.

### Fix B — If you must use the App, calibrate tip length for the custom rack

If you need the App workflow, you must complete a **tip‑length calibration** for the custom
rack before its geometry is trusted: *Robot → Calibration → Tip Length Calibration* (or the
Labware Position Check flow), select the `byu_color_sensor_charging_port` rack and the
left‑mount `p20_single_gen2`, and complete the guided jog. This stores a calibrated length
that overrides the nominal `tipLength − 8.25` fallback (see `tip_handler.py` above) and
removes the 8.25 mm geometry shift.

If the *calibration flow itself* errors out, fall back to Fix A, check Fix D (deck/Z
envelope), and re‑seat / re‑print a straight sensor body — a bent or stretched sensor changes
the real length and defeats any calibration.

### Fix C — Compensate for the ignored overlap directly in the JSON (no‑calibration workaround)

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
matching the physical sensor even when no tip‑length calibration is present. (The AC reference
instead uses `p300_single_gen2`, whose `default` overlap differs — always re‑derive against
the pipette you actually load; see `shared-data` `tipOverlap.default`.)

### Fix D — Watch the Z / deck envelope when carrying an ~84 mm "tip"

An 84 mm "tip" plus a target labware can exceed the OT‑2's usable Z travel and trigger an
out‑of‑bounds / deck‑conflict error that *looks* like a calibration failure. When moving the
sensor over other labware (e.g. the `corning_96_wellplate_360ul_flat` in
`protocol_pick_move_return.py`), keep generous positive `z` offsets on `.top()` moves and
avoid commanding the tip critical point below a well's bottom. Prefer explicit
`plate["A1"].top(z=...)` / `.bottom(z=...)` targets rather than relying on the (overlap‑
adjusted) tip critical point.

## Checklist for the BYU protocols

1. Prefer the AC reference execution path: run on the robot via `opentrons_execute` / Jupyter
   / Prefect with programmatic `pick_up_tip`, and **don't** run the App's tip‑length /
   Labware Position Check calibration for the WCS rack (Fix A). To stay headless without SSH,
   drive the same commands over the robot's HTTP API (Fix A‑HTTP).
2. Load the `p20_single_gen2` on the **left** mount (already corrected in PR #116). Note the
   AC reference uses `p300_single_gen2` on the **right** mount.
3. If you do use the App, perform **tip‑length calibration** for
   `byu_color_sensor_charging_port` on the robot (Fix B).
4. If you cannot calibrate, set the labware `tipLength` to `92.25` for `p20_single_gen2`
   (Fix C) so the fallback geometry is correct.
5. Re‑verify the two carried‑over offsets after the tip model is corrected — the `−1.3 mm`
   measurement offset (how far below the target well top the sensor is lowered to take a
   reading, used in `protocol_pick_move_return.py`) and the `−80 mm` return‑drop offset (how
   far the sensor is lowered back into its charging port on return). Both were tuned against
   the AC reference geometry and may shift by up to the 8.25 mm error once the model is right.
6. Keep generous `z` clearance on transit moves (Fix D).
