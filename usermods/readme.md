# Custom usermods

The bundled usermods have been removed from this fork. This directory remains
available for your own add-on. WLED's usermod manager, registration API, build
loader, and shared usermod CI configuration remain available.

## Build configuration

This fork builds only `esp32c6dev_8MB` (ESP32-C6, 8 MB flash). No usermods
are enabled by default, and no override is needed for a fresh checkout.
For optional local settings, copy `platformio_override.sample.ini` to
`platformio_override.ini` in the repository root; this file is ignored by Git.

Build with `npm ci`, `npm run build`, then `pio run -e esp32c6dev_8MB`.
Flash with `pio run -e esp32c6dev_8MB --target upload`.
The default USB setting uses a USB-to-UART bridge. The override sample shows
how to select native USB Serial/JTAG if that is the connector you use.

Create your module in `usermods/<name>/`, with a `library.json` containing
`"build": {"libArchive": false}` and C++ sources using `REGISTER_USERMOD`.
Add `custom_usermods = <name>` under your selected `[env:<board>]` section in the
local override. The loader is implemented in `pio-scripts/load_usermods.py`.
Allocate GPIO pins through `PinManager` before using them.

## Custom dashboard

WLED already checks the device filesystem for `/index.htm` before serving its
embedded main page (`wled00/wled_server.cpp`, `handleStaticContent` and the `/`
route). Upload your custom dashboard as `/index.htm` to replace the main control
page. The first-start welcome/settings flow still applies until initial setup
is completed. The embedded page remains the fallback when that file is absent.

For a dashboard embedded in the firmware instead, edit `wled00/data/index.htm`
and its assets, then run `npm ci` and `npm run build`. Do not edit generated
`wled00/html_*.h` or `wled00/js_*.h` headers. Deleting usermods alone does not
replace the current dashboard.

The shared `platformio_override.usermods.ini` supports CI for future modules;
it does not contain a usermod implementation.
