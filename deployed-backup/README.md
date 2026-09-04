# deployed-backup

Verbatim snapshots of the live Home Assistant config for the two predictors
(`wasmachine`, `vaatwasser`), taken **2026-09-04** right before the
"coarse/fine alignment + mean-error scoring" matching fixes were applied.

Restore = paste the `state` back into the template helper (Settings → Devices &
Services → Helpers → the sensor → edit), or re-import the automation JSON.

| File | Entity |
|---|---|
| `wasmachine_resterende_tijd.helper.json` | `sensor.wasmachine_resterende_tijd` |
| `wasmachine_match_curve.helper.json` | `sensor.wasmachine_match_curve` |
| `vaatwasser_resterende_tijd.helper.json` | `sensor.vaatwasser_resterende_tijd` |
| `vaatwasser_match_curve.helper.json` | `sensor.vaatwasser_match_curve` |
| `wasmachine_sessie_einde.automation.json` | `automation.wasmachine_sessie_einde_geschiedenis_bijwerken` |
| `vaatwasser_sessie_einde.automation.json` | `automation.vaatwasser_sessie_einde_geschiedenis_bijwerken` |

`input_text.wasmachine_sessie_curves_grof` before repair (6 slots, desynced
from the 8 fine curves):

```
0.36|0.38|0.39|0.41|0.43;0.09;0.47|0.49|0.5|0.52|0.56|0.63;0.25;0.69|0.72|0.74;0.3|0.33|0.35|0.38
```
