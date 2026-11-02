# inter_comms

Encrypted group communications for the fixed ESP32-C6 glass-picture controller.
This v2 usermod follows the registration pattern documented in `AGENTS.md`;
the original `EXAMPLE` usermod is absent from this fork.

The backend owns discovery, peer authentication, automatic master election,
lighting synchronization, shared settings, and firmware distribution. The
separate [`inter_comms_ui`](../inter_comms_ui/readme.md) usermod owns the dashboard,
public HTTP handlers, and generated web assets. They communicate
through `InterCommsService` in `inter_comms.h`; HTTP callbacks never directly
modify WLED state. Both modules execute state/configuration/OTA work on WLED's
main loop. A bounded FreeRTOS mutex protects the UI's cross-task request slots.

## Provision and build

1. Run `python3 usermods/inter_comms/provision.py` once for a new group.
   It creates `group_keys.h` and `private_credentials.json` with independent
   random 256-bit device-group and image-profile material. Both files are ignored
   by Git and written with mode 0600. The script does not print the keys.
2. Enable both modules in the ignored local `platformio_override.ini`:

   ```ini
   [env:esp32c6dev_8MB]
   custom_usermods = inter_comms inter_comms_ui
   ```

3. Run `npm ci`, `npm run build`, then `pio run -e esp32c6dev_8MB`.
4. USB-flash the **same keyed application** to each group member:
   `pio run -e esp32c6dev_8MB --target upload`.
5. Open the device IP in a browser. The dashboard connects automatically, with
   no key, password or login. For initial Wi-Fi setup, connect to the device's
   AP, save Wi-Fi through the dashboard, and reboot the device.
6. When connected to the same LAN, devices discover one another automatically.
   A follower dashboard offers a button to connect to the elected master.

Rebuilds reuse the existing keys. Never run provisioning again for an existing
group unless you intend to replace it. To move the build to another workstation,
securely transfer the private credentials file and run:

```sh
python3 usermods/inter_comms/provision.py --import-credentials /private/path/credentials.json
```

`--rotate` explicitly replaces existing files. All group members must then be
USB-reflashed together. New builds without keys retain a closed control surface;
there is no default key or unauthenticated pairing mode. Do not share keyed
firmware publicly: keys are embedded in the application, as requested.

## Group behavior

- Same-LAN IPv4 discovery uses UDP port **28126**. AP isolation/firewalls must
  permit device-to-device UDP and subnet broadcasts; this is not a mesh router.
- Each device supports up to **16 peers**. The lowest authenticated, live
  station MAC becomes master. Startup election takes five seconds, heartbeats
  run every two seconds, and peer leases expire after eight seconds.
- Discovery does not establish trust. A peer must answer a fresh random
  challenge bound to the receiver's current boot before it affects election.
- Elections converge after network partitions heal. A partition can have its
  own master; this is deterministic failover, not quorum consensus.
- Master snapshots carry brightness, colors, segments, effects, palettes, and
  the effect timebase. Local button changes on a following device are proposed
  to the master, which republishes the resulting group state.
- Periodic snapshots repair dropped state messages. Clocks have LAN delay and
  jitter; random/independently seeded effects are not guaranteed pixel-identical.
- State snapshots are bounded to 3072 serialized bytes. Very large segment
  configurations are not transmitted. Individual reliable commands are bounded
  to 1023 serialized bytes; normal dashboard lighting/settings fit that limit.
- Set **Follow group effects** off on a device before controlling its lighting
  independently. Group-wide explicit commands still apply to selected devices.
- Shared settings are an explicit allowlist: name, follow-group flag, boot
  on/off, boot brightness, and transition duration. GPIO/LED hardware, secrets,
  arbitrary files, presets, and executable JSON commands are not copied.
- Temporary `calibration` commands accept `body.whiteReduction` from 0 to 100.
  This removes the common RGB component and attenuates explicit white at output,
  across all effects. It does not change synchronization or persist to flash;
  reboot or a value of 0 restores normal output. Heartbeats report the current
  value; missing telemetry indicates older firmware without this capability.
- Remote settings/lighting commands retry six times, with acknowledgements or
  timeout visible in the device table. A successful queue response is not proof
  that a remote device has applied a command.
- Wi-Fi provisioning and reboot currently apply to the directly connected
  device. Devices retain individual Wi-Fi credentials and other private settings.

## Firmware sharing

On the elected master, choose a **Firmware destination** and click **Send this
device's firmware**. The device incrementally hashes its running application,
then sends that application directly from its flash partition. There is no
browser file upload or temporary filesystem copy for this path. Preparation
progress remains visible. The application length comes from ESP-IDF image
metadata, including the appended image hash, rather than the partition capacity.

Alternatively, expand **Upload a different application image**, select a compatible
application `.bin`, and stage it. This path needs filesystem space for the file
plus 64 KB reserve. It checks compatibility metadata, size, complete SHA-256 and
keyed module identity. Upload errors are returned explicitly, and the browser
cancels its incomplete upload after a failure so another attempt can start.

**Send staged image** sends the selected image to the chosen follower(s).
Matching build IDs are skipped. Different builds with the same WLED version
label remain distinguishable: the short build ID is derived from the ELF hash
inside the ESP application descriptor. A differing ID is not a chronological
version comparison; choose the intended destination/image explicitly.

The firmware-update table is independent of the general device overview and
shows each device's version/build, state, byte progress and result/error:

- Preparing image / waiting for device.
- Transferring, including byte offset and percentage.
- Verifying image / waiting for reboot.
- Build verified after reboot, or a persistent error.

An acknowledgement only means the offer was accepted. Completion requires a
new authenticated boot epoch and the expected build ID in a subsequent heartbeat.
A reboot into another build is reported as a failure. Older firmware without
telemetry is labelled accordingly; its reported errors remain visible, and its
lack of progress/build confirmation is not treated as success. Later Idle
heartbeats do not overwrite an update failure. A disconnected update target stays
in the update table with an offline flag and timeout reason instead of disappearing.

Followers pull authenticated chunks with offset checks and retries, verify full
SHA-256/group identity and use platform OTA validation before switching slots.
No progress or confirmed reboot for 30 seconds is reported as a failure. A master
change aborts a receiving transfer. Update followers first; the master's local
install action is blocked while follower updates are outstanding. A running
image cannot be replaced during active follower transfers.

An older device with the release-name termination bug may reject a matching
image before it can install this fix. Install the corrected build over USB once
if that happens. The regression test poisons automatic storage to reproduce the
old 12-character release string becoming a 48-character string.

Image sources/transfer tracking are not restored after a master reboot; prepare
or stage the image again on the current master. Physical flash power-loss and
rollback behavior still require hardware validation.

ESP-IDF API reference used for running-image length and hash handling:
[Espressif image format implementation](https://github.com/espressif/esp-idf/blob/v5.5.1/components/bootloader_support/src/esp_image_format.c).

## Security boundary

Device messages use **AES-256-GCM**, including discovery, heartbeats, state,
settings, firmware offers and chunks. SHA-256 is a hash, not encryption. HMAC-SHA256
derives a sender key from the group key, a domain label, station MAC and random
128-bit boot epoch. A monotonically increasing 32-bit sequence supplies the GCM
nonce. Exhausting the counter stops transmission instead of reusing a nonce.
Authenticated headers bind sender, message kind, receiver boot epoch, sequence,
and payload length. Receiver challenges and per-peer sequence checks reject
captured packets from previous boots and already consumed sequences.

The dashboard is intentionally **public**, as requested for this project. Anyone
who can reach the page can use its allowed lighting, settings, Wi-Fi setup,
reboot and firmware-update commands. The URL does not authenticate a visitor.
Browser requests/replies use ordinary JSON over HTTP, including Wi-Fi setup
credentials; this channel has no application encryption or dashboard password.
Public one-use request IDs only correlate bounded queue slots and suppress
accidental duplicate submission. Device-to-device authentication/encryption is
independent and remains enabled.

The private credentials JSON retains its `adminKey` field for compatibility with
the existing firmware image-profile fingerprint. It is no longer a browser key,
not exposed through HTTP, and never needs to be entered during setup. Existing
group/profile provisioning files can be reused without rotating keys.

An exclusive-network-control capability in the WLED core gates stock UDP sync,
realtime E1.31/Art-Net/DDP, MQTT input, Alexa, Hue and ESP-NOW control. The UI's
catch-all handler gates stock HTTP/JSON/WebSocket/file-editor/OTA endpoints.
The backend installs a closed HTTP gate if the UI is absent. Existing external
WLED integrations therefore cannot control an enabled device. USB serial/local
buttons remain available. NTP, DHCP and other network infrastructure are outside
this encrypted group protocol.

All keyed group members are trusted: a compromised group member can impersonate
another member. This is not per-device public-key identity or manufacturer-signed
firmware. Keys are extractable by someone who obtains a keyed image or sufficient
physical access; ESP secure boot/flash encryption are separate provisioning work.
The protocol does not prevent radio jamming, traffic analysis, or resource-exhaustion
attacks. Avoid claiming deployment validation until the real-device tests below run.

## Source layout and validation

- `inter_comms.cpp`: backend state machine and command/firmware service.
- `inter_comms.h`: interface used by the separate UI module.
- `protocol.h`: portable frame, admission, replay, election and image matcher logic.
- `crypto.h`: Mbed TLS adapter, key derivation and bounded encoding helpers.
- `provision.py`: private build-time provisioning.
- `tests/`: native C++ boundary/replay/election tests plus crypto/provisioning tests.

`npm test` includes the module tests. The C++ protocol tests use a host `c++`
compiler with AddressSanitizer and UndefinedBehaviorSanitizer. The actual public browser client is tested over HTTP without credentials.
The separate simulator tests check malformed JSON, request reuse, keyless
settings access, and the unchanged encrypted group protocol. Firmware compilation validates ESP32-C6 linking and module registration.
These checks do **not** verify radio behavior or OTA recovery on physical devices.

Before deployment, test at least two ESP32-C6 devices: independent provisioning
rejection, discovery, master power loss and rejoin, isolated/rejoined network
partitions, local button and effect synchronization, interrupted firmware transfer,
power loss during OTA, wrong-board/wrong-key/corrupt image rejection, and a complete
follower-first rollout. Verify no legacy endpoint/protocol can change the LEDs.

API references used for cryptographic integration:
[Mbed TLS GCM](https://mbed-tls.readthedocs.io/projects/api/en/development/api/file/gcm_8h/),
[Espressif security overview](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c6/security/security.html),
[noble-hashes](https://github.com/paulmillr/noble-hashes).

### Passive simulator discovery and firmware management

The desktop simulator connects discovery and encrypted firmware transport on startup.
With lighting control disabled it advertises `canLead: false`, remains visible as an
authenticated peer, and does not participate in lighting control or master election.
Enabling lighting control makes it eligible for the normal lowest-identity election.

Firmware offers are accepted from authenticated group peers independently of the
lighting master. Each transfer is bound to its admitted source and boot epoch;
changing the lighting master does not interrupt it. A source disconnect or reboot
aborts the transfer. Lighting/settings commands still require the elected master.
All participating devices need the matching firmware for these semantics.

### Automatic distribution of newer releases

From project release **1.0.1**, physical devices automatically offer their running
application to older authenticated peers. No lighting master role or dashboard action
is required. The source waits 15 seconds after boot and checks every five seconds.
The newest stable `major.minor.patch` release wins; equal releases use the lowest MAC
to select one distributor. Equal-version rebuilds and unknown/development version
labels require manual distribution. Automatic offers never intentionally downgrade.

The source hashes its running application incrementally and updates one peer at a
time. Manually staged images are left for manual distribution. Existing AES-GCM,
SHA-256, board/release compatibility and keyed group-profile checks remain in force.
Receivers running 1.0.1 also reject automatic offers with an equal/older release label.
Success requires an authenticated reboot reporting the expected build fingerprint.

Failed targets retry at five-minute intervals, up to three automatic attempts per
target during the distributor's current boot/membership record. Status and failures
remain visible in the firmware table. A receiving device needs `peerUpdates` support
(available in project 1.0.0); older unsupported devices require manual bootstrapping.

Install 1.0.1 or a later release on one board first. It will distribute to older
compatible boards when they join. The desktop simulator is excluded by default;
explicit automatic-update participation makes it eligible without enabling lighting
control. Discovery is partition-local, so disconnected groups coordinate separately.
