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
- `web/model-efr12.png`: matching mask from `Glasbild_LillE.stl`
  (2,997,509 triangles), displayed as EFR12 LilliE.
- `web/model-efr15.png` and `web/model-efr14.png`: matching masks from
  `EFR15_Glasbild_2023.stl.zip` (1,335,040 triangles) and `EFR14_Glasbild.stp`
  (6,060,055 tessellated triangles). The STEP reader reported one unresolved
  reference; the imported side view was visually inspected, but completeness
  against the original CAD assembly has not been established.
- `build_ui.mjs`: bundles the adapter and minifies the page with gzip level 9.
  The model is served separately at a content-hashed URL with immutable caching,
  so controls can start before the selected image downloads. HTML and all four
  masks share a 768 KiB flash asset budget, increased from 256 KiB for one car.
  Firmware and offline previews share 1280 x 800 WebP masks, resized with
  Lanczos filtering and quantized to 16 alpha levels before lossless WebP
  encoding. The original 1600 x 1000 PNG sources remain available for
  regeneration. Framing and lighting parameters are shared by all four models.
- `inter_comms_ui.cpp`: bounded main-loop RPC queue and flash-resident HTTP page.

`dashboard.html` and `dashboard_generated.h` are ignored, generated artifacts.
Opening `dashboard.html` directly as a local file uses the offline simulation and the selected `web/model-*.webp`; device-hosted pages use the live adapter.
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
applying a preset sends the complete lighting state. The effect dropdown loads the running firmware's `/json/effects` catalog and uses WLED IDs directly, including custom effects.
The offline preview uses the pinned [`@wiillownet/wled-fx-sim` 0.6.0](https://github.com/wiillownet/wled-fx-sim) browser ports of WLED v16.0.0 (EUPL-1.2), not animations inferred from effect names. Effect IDs, palette defaults, custom sliders and checkbox defaults are extracted from this checkout's `wled00/FX.cpp` and `FX.h` by `build_effects.mjs`; see `Segment::setMode` in `FX_fcn.cpp`. The port advances persistent effect state at WLED's 23 ms frame interval. Each preset and the main view own separate simulators.

The offline base is a **1D strip**, initially 30 LEDs (`DEFAULT_LED_COUNT` in `wled00/const.h`). Set **Preview LEDs** to the actual strip length. The narrow LED row shows each raw pixel; glass shading remains an optical approximation. Matrix-only effects use the firmware's Solid fallback on a 1D segment instead of projecting invented matrix animations. Image and Copy Segment explicitly report missing file/segment inputs. Slow Transition (219), newer than the dependency, implements the local source's solid-primary palette transition, including its 10-second minimum and minute-based speed setting. Audio ports use the library's synthetic audio fixture, not a microphone. The dependency targets perceptual similarity, not bit-for-bit parity with this development checkout; RNG, audio input and firmware-version differences still apply.

`web/effect-preview.js` adapts the port; `web/effect-preview.generated.js` and `web/effect-preview.NOTICES.txt` are generated by `npm run build`. The latter retains the upstream EUPL/MIT licenses and palette notices. Offline packages and the readable generated dashboard embed the engine and full notices. The device-served document excludes that browser-only bundle and continues to use `/json/live`; non-selected animated preset thumbnails stay neutral until selected for live output. The standalone offline package has a separate 1 MiB compressed budget; the firmware's four-model asset budget is 768 KiB.

`node --test usermods/inter_comms_ui/tests/effect-preview.test.mjs` checks source-derived Blink timing, Wipe/Sweep direction and boundaries, Scan width/position, Colorloop versus Rainbow, matrix fallbacks, Slow Transition, defaults, power/brightness and independent state. All local IDs are also checked for valid pixel frames or an explicit missing-input result. This does not establish hardware or optical parity.

Existing three-choice browser presets are migrated to WLED IDs. The connected main model
renders WLED's actual `/json/live` LED colors directly, with no local effect
prediction or interpolation. Requests run at up to ten frames per second,
never overlap, and pause in hidden or off-screen views. Initial assets finish
before requests start; failures retry after ten seconds and clear the model.
The selected device supplies the feed (the first listed device for All bases).
Colors already include brightness. Physical
light scattering remains an approximation. Offline preset cards use the same ported effect bodies as the main view.

Explicit dashboard lighting commands include `independent: true`. On successful
application, the backend disables that target's periodic group synchronization
so individual settings are not overwritten by the master's next snapshot.
All bases sends explicit group commands. Follow group effects in Settings
can re-enable synchronization; subsequent explicit lighting edits opt out again.
Telemetry from older firmware may omit color, speed and intensity; update all
group members to obtain complete state readback.

Each device has five independent named presets storing color, brightness, power,
effect, speed and intensity in the browser, not in firmware. All bases has its own
five slots; applying one sends that lighting state to every device. Device slots
follow stable device IDs, including after discovery order or name changes. Existing
shared presets seed each new set of slots. Saving a preset does not send commands
or overwrite another target's slots.
Speed and intensity display as percentages while commands retain WLED's byte
range. The sun/moon toggle remembers the browser's theme preference.

Settings preserves Wi-Fi setup, reboot, shared startup settings, firmware
staging, sharing the running firmware, follower rollout and local installation.
Wi-Fi credentials apply only to the connected device and are cleared from the
password input after a successful save. Firmware transfer state and errors
remain visible in Settings.

## Resource use and access

**Model** in Settings > General lists EFR12 LilliE, EFR14 HerminE, EFR15 RosE,
then EFR16 DaisiE for the main view and
preset thumbnails. Choices are stored in this browser per stable device ID;
All bases has a separate choice. This is visual configuration and sends no
lighting or firmware configuration commands. Native lettering is retained.

### Rebuilding car masks offline

`prepare_models.py` requires Python 3.12 with `numpy`, `pillow`, and
`cadquery-ocp` 8.0.1.1.0 for STEP. `render_models.cjs` requires Playwright;
set `GLASSLIGHT_PLAYWRIGHT` to its module location and `GLASSLIGHT_BROWSER`
to an installed Chromium/Edge executable when using an existing installation.

```powershell
py -3.12 prepare_models.py <source.stl.zip-or-source.stp> <model.bin>
node render_models.cjs <model.bin> <model.alpha>
py -3.12 prepare_models.py <model.alpha> web/model-efr15.png --alpha
py -3.12 prepare_models.py web/model-efr15.png web/model-efr15.webp --compact
py -3.12 prepare_models.py web/model-efr12.png web/model-efr12.webp --compact
```

The converter centers the native X/Z side view and fits the same 88% glass
width as EFR16. STEP tessellation uses 0.05 model-unit linear and 0.15 radian
angular deflection. The renderer preserves the original 3200 x 2000 WebGL
surface shader, transparent hidden surfaces, 4800-cell stippling, 0.22 dot
threshold, and 0.48–0.72 alpha, then downsamples to a 1600 x 1000 PNG with
16 alpha levels. The compact step resizes that master to 1280 x 800, requantizes
to 16 alpha levels and writes a WebP with lossless encoding. Use `--width 1600`
to retain the original dimensions. CAD files and intermediate meshes stay outside firmware.

### Browser rendering and device access

Main and thumbnail rendering runs in the browser at approximately 24 and 12
fps. Scaled masks and static cards are cached; off-screen and hidden-page
animations stop. Reduced motion is respected. Actual ESP32-C6 heap, network
concurrency and LED timing still require hardware measurements.

The dashboard has no login. Anyone with network access to the device can use
its allowed controls. HTTP does not encrypt browser commands or Wi-Fi setup;
device-to-device communications retain the existing authenticated encryption.
Private group keys never enter the HTML or browser APIs. The UI catch-all
allows read-only GET `/json/live` and `/json/effects` and blocks other legacy API/file/OTA routes; GPIO configuration is absent.
