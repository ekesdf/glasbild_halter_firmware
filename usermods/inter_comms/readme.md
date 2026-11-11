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

## Reading the backend

Start with `InterComms::loop()` in `inter_comms.cpp`. It initializes the boot
session, services network input, updates the radio channel and leader, publishes
lighting state, and advances firmware work in that order.

| File | Responsibility |
| --- | --- |
| `inter_comms.h` | Small service interface used by the dashboard adapter. |
| `inter_comms_internal.h` | Private class declaration and shared state, including peer tables, buffers and transfer progress. |
| `inter_comms.cpp` | WLED lifecycle hooks, readiness, main-loop scheduling and usermod registration. |
| `inter_comms_transport.cpp` | UDP/ESP-NOW input, authenticated admission, message dispatch, telemetry, channel selection and master election. |
| `inter_comms_commands.cpp` | Lighting/settings validation, dashboard operations, reliable command retries and state synchronization. |
| `inter_comms_firmware.cpp` | Image staging, running-image preparation, verified OTA installation and manual/automatic distribution. |
| `radio_transport.h`, `radio_transport.cpp` | Bounded ESP-NOW queues and callbacks; one shared callback owner is defined in the source file. |
| `protocol.h`, `radio_protocol.h`, `crypto.h` | Frame/admission rules, radio fragmentation/election helpers and cryptography. |

The implementation files define methods of one backend instance. They share its
existing state and main-loop scratch buffers; splitting the source does not add
worker tasks or independent copies of the peer tables. HTTP work still enters
through `InterCommsService` on the main loop.

For incoming messages, follow `processPacket()` first: it validates the frame and
receiver boot epoch, authenticates GCM, performs challenge/proof admission, and
checks replay protection before `processAuthenticatedPacket()` dispatches the
payload. `updateMemberFromHeartbeat()` handles optional telemetry and verifies
the expected firmware build after a peer reboots.

For updates, distinguish the **source image** from the **incoming OTA transfer**.
`stage()` verifies a filesystem upload; `prepareCurrent()` hashes the running
application without copying it. `distributeFirmware()` offers that source to
followers or selects a local installation. `writeFirmware()` validates
compatibility before writing, then checks the complete digest and group profile
before activating the OTA image. `serviceCommands()` retries the offers, while
`serviceTransfers()` advances chunks and enforces progress timeouts.

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

- Devices discover one another directly using **ESP-NOW**, with same-LAN IPv4
  UDP port **28126** retained for older devices and the desktop simulator.
  A successful receiver challenge promotes an existing UDP peer to ESP-NOW.
- Each device supports up to **16 peers**. The gateway with the strongest Wi-Fi
  signal becomes master. Signal strength is grouped into 5 dB bands; equal bands
  use the lowest station MAC. A better live candidate must remain preferred for
  six seconds before handover. A disconnected gateway yields immediately when
  another eligible peer is known. A fallback AP with a browser client can lead
  when no LAN-connected gateway is available.
- With no reachable LAN/AP gateway, hardware devices still elect a radio-only
  master by MAC so local group control can continue.
- Startup election takes five seconds, heartbeats run every two seconds, and
  peer leases expire after eight seconds. All boards need this firmware for the
  new election rules; older builds still elect by MAC alone.
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
- Dashboard lighting edits automatically opt the selected device out of periodic
  master snapshots. Choose **All bases** to send explicit lighting edits to every
  base, or select one base to edit it independently.
- Shared settings are an explicit allowlist: name, follow-group flag, boot
  on/off, boot brightness, and transition duration. GPIO/LED hardware, secrets,
  arbitrary files, presets, and executable JSON commands are not copied.
- Remote settings/lighting commands retry six times, with acknowledgements or
  timeout visible in the device table. A successful queue response is not proof
  that a remote device has applied a command.
- Wi-Fi provisioning and reboot currently apply to the directly connected
  device. Devices retain individual Wi-Fi credentials and other private settings.

### Direct dashboard communications

Open either device's dashboard on the router network. A follower automatically
opens the elected gateway after startup/election settles and the destination
confirms that it is still master. Browser history is replaced to avoid back-button
loops. An unreachable or changing master is retried by normal status polling;
the follower page remains available until a confirmed redirect is possible.
Discovery, device telemetry, reliable
lighting/settings commands, effect snapshots, firmware messages and remote LED
previews can travel directly between boards over ESP-NOW. The browser relays a
new follower's preview through the gateway instead of fetching the follower's IP.
The device list identifies ESP-NOW links and reports each device's Wi-Fi RSSI.
The browser still needs HTTP connectivity to one gateway over LAN or its fallback
AP; a browser cannot speak ESP-NOW itself. This is a single-hop direct link, not
a mesh router or an Internet bridge.

ESP-NOW uses the current Wi-Fi channel while the station is connected. After
30 seconds without a station connection, a disconnected board searches channels
allowed by the radio's country configuration with a randomized four-to-six-second
dwell and holds the channel once an authenticated direct
peer is available. Background Wi-Fi reconnect scans pause while this radio mode
is active. A board with an AP client holds its channel to preserve that browser
connection. If no reachable gateway is known, 90-second radio search windows
alternate with 30-second router connection windows, so restoring a router does
not require a reboot. Rebooting also gives LAN association a fresh opportunity.
The core `getWiFiChannelRequest()` usermod capability arbitrates this radio use;
other usermods return zero and retain normal reconnect behavior.

The existing AES-256-GCM envelope, receiver challenge, boot binding and replay
checks apply unchanged over the direct link. Oversized ciphertext envelopes are
fragmented into packets of at most 250 bytes and reassembled in fixed-size,
expiring slots. Discovery is broadcast; established peers use unicast with MAC
acknowledgements and up to three attempts per fragment. Application acknowledgements
still determine whether a control command was applied.
Only a fully reassembled and authenticated frame affects device state. Wi-Fi-task
callbacks only copy bounded packets into a FreeRTOS queue; processing and sends
run on WLED's main loop. The core `onNetworkReset()` lifecycle callback stops
ESP-NOW before the shared Wi-Fi driver is restarted. Remote previews sample up
to 256 actual output pixels.

Implementation references: [Espressif ESP-NOW API and callback guidance](https://docs.espressif.com/projects/esp-idf/en/v5.5.1/esp32c6/api-reference/network/esp_now.html).
The crypto adapter supplies a valid input pointer for empty messages because
[Espressif's AES-GCM implementation](https://github.com/espressif/esp-idf/blob/v5.5.1/components/mbedtls/port/aes/esp_aes_gcm.c#L448-L475)
requires one even for zero-length data.

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
