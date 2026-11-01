# inter_comms_ui

The web-interface usermod for [`inter_comms`](../inter_comms/readme.md).
Enable **both** modules using `custom_usermods = inter_comms inter_comms_ui` in
`platformio_override.ini`. The local override is ignored by Git. The UI depends
on the backend's C++ service interface; it is not a standalone communications
module and does not implement its own peer/election/update logic.

Editable files:

- `web/index.html`: dashboard structure and CSS.
- `web/app.js`: browser interaction and public JSON RPC client.
- `web/updates.js`: running-image sharing, destination selection and per-device update feedback.
- `web/upload.js`: file staging with progress and failed-upload cleanup.
- `web/request.js`: keyless JSON RPC with bounded, single-use request IDs.
- `inter_comms_ui.cpp`: HTTP endpoints, request bounds, public browser
  commands, and the cross-task work queue.
- `build_ui.mjs`: bundles the browser code and embeds the compressed page.
- `build.py`: PlatformIO integration for the dashboard build.

Run `npm ci` at the repository root, then `npm run build`. The PlatformIO library
build also runs the UI generator. `dashboard_generated.h` and `dashboard.html`
are generated, ignored artifacts: edit `web/` instead. `dashboard.html` can also
be opened locally as a trusted standalone administration interface.

The dashboard opens automatically with **no key, password or login**. Anyone who
can reach its HTTP page can view and change the allowed lighting/settings, set
Wi-Fi, reboot, and stage/distribute compatible firmware. The page URL is not an
access-control mechanism. Browser commands and Wi-Fi setup travel over ordinary
HTTP; device-to-device messages remain authenticated and encrypted with the
private group key embedded during flashing. Group/profile keys are never sent
to a browser or required from an end user.

Public request IDs reserve bounded queue slots; they are not credentials.
Requests still run on WLED's main loop, with backend validation and hardware
restrictions intact. See the backend readme for provisioning, allowed settings,
group behavior, firmware rollout and hardware validation requirements.

The dashboard owns a catch-all HTTP handler. Once enabled, this replaces the
stock WLED web interface and prevents direct legacy API/file/OTA bypasses.
Initial Wi-Fi setup is available inside the public dashboard. GPIO and LED
hardware settings are intentionally absent.

The device-hosted page hides its connection section, device list and target
selector unless it is the elected master and at least two physical devices are
identified and online. Simulator entries are excluded from both the count and device list;
legacy simulator versions ending in `(sim)` are also recognized. The threshold
includes the local device when it is physical. A standalone local HTML file
shows its initial address input only until it has connected to a device.
