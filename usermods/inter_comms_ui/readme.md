# inter_comms_ui

Glasslight's device dashboard for the [`inter_comms`](../inter_comms/readme.md)
backend. It replaces the old dashboard at `/` and `/inter_comms`. Enable both
modules in the ignored local `platformio_override.ini`:

```ini
[env:esp32c6dev_8MB]
custom_usermods = inter_comms inter_comms_ui
```

Reuse the existing `usermods/inter_comms/group_keys.h` for your device group.
Do not regenerate credentials for an existing installation. Build with
`npm ci`, `npm run build`, then `pio run -e esp32c6dev_8MB`.

## Sources and generated files

- `web/preview.html`: shared layout, styling, browser simulation and preset UI.
  Open directly for the standalone demonstration using sample devices.
- `web/app.js`: live-device adapter, debounced lighting commands and management.
- `web/request.js`: public single-use RPC reservation client.
- `web/devices.js`: physical-device filtering.
- `web/updates.js`, `web/upload.js`: firmware management helpers.
- `web/model-mask.png`: 198 KB, 1600 x 1000 indexed PNG generated offline from
  all 2,475,105 triangles in the supplied `Glasbild_final_prt_efr16 v1.stl`.
- `build_ui.mjs`: bundles the adapter and minifies the page with gzip level 9.
  The model is served separately at a content-hashed URL with immutable caching,
  so controls can start before the image downloads. Both assets share a 256 KiB budget.
- `inter_comms_ui.cpp`: bounded main-loop RPC queue and flash-resident HTTP page.

`dashboard.html` and `dashboard_generated.h` are ignored, generated artifacts.
The firmware serves the compressed array directly from flash; it does not
decode the image or render the STL. The old `web/index.html` is removed.
`build_preview.mjs` optionally packages the disconnected demonstration into
`build_output/inter-comms-preview/`.

## Live controls

Device-hosted pages use reported device IDs, names and lighting telemetry,
not browser-stored demonstration state. A follower page shows its local state
and offers an Open master action in Settings. Lighting commands are issued
from the settled master. Discovery/status reads never write lighting state.

Slider changes are coalesced for 250 ms; RPC operations are serialized.
Status refreshes every thirty seconds, or every three seconds while Settings is
open, and pauses in hidden tabs. Group edits send only the edited properties;
applying a preset sends the complete lighting state. Solid, Breathe and Color
Drift map to WLED effect IDs 0, 2 and 9 respectively. The connected main model
renders WLED's actual `/json/live` LED colors directly, with no local effect
prediction or interpolation. Requests run at up to ten frames per second,
never overlap, and pause in hidden or off-screen views. Initial assets finish
before requests start; failures retry after ten seconds and clear the model.
The selected device supplies the feed (the first listed device for All bases).
Colors already include brightness and temporary output calibration. Physical
light scattering remains an approximation. Preset cards and the disconnected
demonstration continue to simulate effects.

White reduction (temporary) adjusts the selected physical device, or All bases,
from 0% (normal) to 100% removal of the common RGB component and explicit white
channel. Colors become stronger while pure white dims. This output calibration
applies to every effect without changing presets or group synchronization.
It is not persisted: set 0% to reset, or reboot the device. Devices without
calibration telemetry require a firmware update before using the slider.

Explicit dashboard lighting commands include `independent: true`. On successful
application, the backend disables that target's periodic group synchronization
so individual settings are not overwritten by the master's next snapshot.
All bases sends explicit group commands. Follow group effects in Settings
can re-enable synchronization; subsequent explicit lighting edits opt out again.
Telemetry from older firmware may omit color, speed and intensity; update all
group members to obtain complete state readback.

Five named presets store color, brightness, power, effect, speed and intensity
in the browser, not in firmware. Saving a preset does not send commands.
Speed and intensity display as percentages while commands retain WLED's byte
range. The sun/moon toggle remembers the browser's theme preference.

Settings preserves Wi-Fi setup, reboot, shared startup settings, firmware
staging, sharing the running firmware, follower rollout and local installation.
Wi-Fi credentials apply only to the connected device and are cleared from the
password input after a successful save. Firmware transfer state and errors
remain visible in Settings.

## Resource use and access

Main and thumbnail rendering runs in the browser at approximately 24 and 12
fps. Scaled masks and static cards are cached; off-screen and hidden-page
animations stop. Reduced motion is respected. Actual ESP32-C6 heap, network
concurrency and LED timing still require hardware measurements.

The dashboard has no login. Anyone with network access to the device can use
its allowed controls. HTTP does not encrypt browser commands or Wi-Fi setup;
device-to-device communications retain the existing authenticated encryption.
Private group keys never enter the HTML or browser APIs. The UI catch-all
allows read-only GET `/json/live` and blocks other legacy API/file/OTA routes; GPIO configuration is absent.
