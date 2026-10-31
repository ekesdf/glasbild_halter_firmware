# Fixed project hardware

`wled00/project_hardware.h` contains the firmware-owned hardware profile:

- 35 WS2812B-2020-V6 RGB LEDs, GRB order, on GPIO 6.
- Estimated full-white current: 12 mA per LED; LED current budget: 420 mA.
- One active-low momentary button on GPIO 4, wired to GND, with internal pull-up.
- No infrared receiver, relay, I2C, or SPI peripherals.

The profile is compiled into the application. On every boot it supplies the hardware
configuration instead of the saved `hw` object. A changed profile is saved back to
`cfg.json` after initialization, including on the first boot after a USB or OTA
firmware update. No filesystem upload or factory reset is needed. Other saved
settings, including Wi-Fi credentials, remain in the normal WLED configuration.

Runtime JSON configuration updates ignore `hw`. The LED/hardware, pin information,
2D, usermod and DMX settings pages reject requests, including form submissions;
the settings menu omits these controls. Button action macros remain configurable
through Time & Macros. This does not disable the corresponding compiled features.

Edit the profile and rebuild the application to change the hardware. The numeric
protocol identifiers are documented beside the JSON and defined in `wled00/const.h`.
The current limiter uses an estimate; verify LED behavior and current on the actual
hardware. The firmware build does not verify electrical wiring or color order.
