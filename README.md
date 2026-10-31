# AsiaHorse Umbra ARGB Hub — Linux Control

Native Linux control for the **AsiaHorse Umbra ARGB hub** (USB `1a86:fe05`,
enumerates as "USBFAN", QinHeng CH55x chip). This tool speaks the hub HID
protocol directly through `/dev/hidraw`. The CLI uses the Python 3 standard library only; the desktop GUI also requires Tk.

Verified working August 2026: full-color control of 124 LEDs across 10
ports on real hardware.

## Quick start

```bash
# one-time: let your user talk to the hub (then replug or: sudo udevadm trigger)
sudo cp 70-asiahorse.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger

python3 asiahorse.py status          # read-only sanity check
python3 asiahorse.py topology        # per-port LED counts
python3 asiahorse.py color 113 230 207
python3 asiahorse.py off
```

## Desktop GUI

A native English-language Linux desktop app with five lighting effects:
**Solid color**, **Breathing**, **Rainbow**, **Color cycle** and **Chase**.
Includes color selection, brightness and speed controls, start/stop, and
read-only temperature and fan-speed monitoring. No pip packages are required.

### Install and launch

Download and extract the entire repository (or clone it), keeping all Python
files together. Install the Tk package for your distribution:

```sh
# Bazzite: install, then reboot before launching
sudo rpm-ostree install python3-tkinter

# Fedora Workstation
sudo dnf install python3-tkinter

# Debian / Ubuntu
sudo apt install python3-tk
```

Run only the command for your distribution. From the project folder, install
the USB permission rule once:

```sh
sudo cp 70-asiahorse.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=hidraw
```

The rule grants access to the active local desktop user via `uaccess`, without
requiring a `plugdev` group. Reconnect the hub USB connection if permissions
have not taken effect. Run the app as your regular user:

```sh
python3 asiahorse_gui.py
```

### Application menu launcher

After installing Tk and the USB rule, run this once as your regular user from
the extracted project folder (no sudo):

```sh
python3 install_desktop.py
```

Search for **AsiaHorse Umbra** in your application menu. The app opens without
a terminal and can be pinned to your dock or favorites. If the menu does not
refresh immediately, sign out and back in.

The installer copies the runtime files into `$XDG_DATA_HOME/asiahorse-umbra`
(default `~/.local/share/asiahorse-umbra`) and installs a desktop entry and icon
under the same data directory. Moving or deleting the downloaded folder will
not break the installed app. Re-run the installer from a newer download to
update it. Close the running app before installing or updating.

To remove the per-user installation, run from a project download:

```sh
python3 install_desktop.py --uninstall
```

This leaves system-wide USB rules and Python Tk installed. Animations still
require the app to remain open; the launcher does not enable autostart.

### Using effects

1. Select an effect and choose a color.
2. Adjust brightness and speed, then press **Start / apply**.
3. Press **Start / apply** again after changing settings.

The selected color applies to solid color, breathing and chase. Rainbow and
color cycle generate their own colors. At 1× speed, one cycle takes four
seconds; speed ranges from 0.25× to 3×. Effects repeat independently along
each port's detected LED chain. All ports share the same settings; physical
fan geometry and separate per-port settings are not configured in this version.

**Stop** freezes the last frame. **Turn off RGB** sends black. Closing the
app stops streaming and leaves the last frame latched. Animated effects require
the app to stay open, streaming at up to approximately 20 frames/second.
Do not run another RGB controller or a second app instance at the same time.
No boot effects or firmware presets are modified.

### Fan curves (experimental)

The **Fan curves** tab adds USB motor duty control alongside RGB. Opening the
app or loading a profile does not change fan duty; press **Apply curve** or
**Apply manual** to take control. This sends a hub-wide duty request, not an
individual setting per port. No motherboard PWM registers are written.

- Choose a temperature sensor, preferably the CPU sensor you intend to follow.
- Edit 2–12 `temperature:duty` points, or drag points on the curve graph. Points
  must increase in temperature and must not decrease in duty. Press **Apply curve**.
- Save named profiles; assigning a digit enables **Ctrl+Alt+1–9 while the app
  has keyboard focus**. These are application shortcuts, not global desktop shortcuts.
  Activating one immediately applies its saved curve and sensor.
- The live graph shows sensor temperature in °C and requested fan duty in %;
  it does **not measure actual fan RPM**. It retains the last 120 samples in memory.
- **Apply manual** requests a fixed percentage. **Stop curve / 100%** replaces
  the curve with a full-duty request; it does not stop the fan motors.

Profiles are stored atomically in `$XDG_CONFIG_HOME/asiahorse-umbra/profiles.json`
(default `~/.config/asiahorse-umbra/profiles.json`). Startup never automatically
activates a saved curve. Changes to an active curve require **Apply curve**;
changes to a saved profile require **Save profile**. Deleting a profile does
not stop its active curve. The desktop installer preserves profiles.

Curve interpolation is linear. Rising requests apply immediately; falling
requests are limited to two percentage points per second. Missing, faulty or
implausible selected-sensor readings request 100%. Any selected-sensor reading
at or above 85 °C overrides the curve to 100%; this is a conservative application
policy, not a universal hardware temperature limit. Control samples once per
second in the USB worker independently of the GUI refresh.

Normal exit attempts to request 100%, leaving software fan control enabled.
It does **not restore the previous firmware curve or motherboard control mode**.
A crash, forced termination, suspend or disconnected USB cannot guarantee a
recovery command: the hub may hold its last duty. A USB error stops curve
control and attempts a 100% request; check cooling and reconnect before applying
again. Manual mode does not follow any temperature sensor.

Supported UI duty range is 30–100%. Only **100% → 75% → 100%** has so far been
hardware-tested by glassknug on their Umbra hub. Lower settings and combined
RGB/curve playback need hardware testing. The starting curve uses 75%, 85%,
100%; do not assume a percentage is a measured speed or that every connected
fan has the same response.

Protocol evidence: static inspection of the user-provided AsiaHorse UMBRA
3.5.5 installer identified the percentage-slider path sending command body
`FB <percentage>` and the fan-control ownership switch sending `08 00/01`.
`08 01` corresponds to motherboard-sync unchecked (software fan control).
The above two-level test confirmed changes in motor speed with `08 01` and
`FB 64` / `FB 4B`. No vendor executable or extracted proprietary code is
redistributed with this project. RGB and fan duty commands share a
single USB worker to avoid competing HID connections inside the app.

### Sensor readings

Temperature and fan speed readings refresh every two seconds from Linux
`/sys/class/hwmon`. Sources and sensor labels are shown alongside readings.
These are system sensors, **not telemetry decoded from the Umbra hub**.
Only sensors exposed by installed kernel drivers appear. A fan connected only
to the hub may have no RPM reading. Missing or faulty readings show
**Unavailable**, while an actual zero RPM reading is shown as zero.
Sensor access is read-only. Fan duty control is separate, via USB in the Fan curves tab.

### Validation and troubleshooting

The Fan curves tab is a new experimental integration. Automated tests cover
curve interpolation, profiles, fallback behavior, USB packets and shared-worker
operation. Its visual layout and combined playback need desktop hardware testing.

glassknug reported all five effects working on an Umbra hub running Bazzite,
and subsequently confirmed that the English GUI works well. Compatibility
with other hardware configurations is not yet established. The new menu
installer is tested with temporary directories; an end-to-end application-menu
launch and the About dialog still need verification on a Linux desktop. This
build environment has no working Tk display or attached hub.

- **Permission denied:** install the `70-asiahorse.rules` rule above and reconnect USB.
- **No hub found:** check the USB data cable and the hub's power connection.
- **No LEDs detected:** check connected lighting devices; the GUI refuses to guess LED counts.
- **No sensors:** only readings provided by your system's hwmon drivers can appear.

Run backend tests without a USB device or desktop:

```sh
python3 -m unittest discover -s tests -v
```

The worker keeps one USB connection open during playback and handles the
hub's software-mode re-enumeration using the existing driver. Tests cover
animation values, packet numbering/padding/checksums, worker lifecycle and
sensor parsing.

## Protocol documentation

Everything below was cross-validated from two independent sources (a
Wireshark capture and a published SignalRGB plugin) plus live testing;
the checksum math reproduces both sources byte-for-byte.

**Transport:** USB HID, 65-byte output reports = report ID `0x00` + 64-byte
payload, zero-padded.

**Checksum:** 8-bit additive over all preceding packet bytes:
`sum = (sum + byte) & 0xFF`, appended as the final meaningful byte.

**Native command packet:**

```
52 42 <len> 00 <body...> <checksum>      len = body length + 5
```

| Body      | Meaning                                          |
|-----------|--------------------------------------------------|
| `00`      | Status query (hub replies with a `52 42` frame)  |
| `01 FF`   | Topology query (reply below)                     |
| `FD 01`   | Enter software-controlled RGB mode (see quirks!) |
| `08 00`   | Motherboard fan-control sync on (3.5.5 UI path)  |
| `08 01`   | Software fan control (3.5.5 UI path)             |
| `FB nn`   | Hub-wide motor duty percentage, `nn` is 0–100    |

**Topology reply:** ten 5-byte records starting at offset 6:
`[led_count, ?, ?, port_index, ?]` for ports 0–9.

**Direct RGB streaming packet:**

```
88 <packet_count> <packet_index> <20 x R,G,B> <checksum>
```

Frame = all ports' LEDs concatenated in port order 0–9, padded with black
to `packet_count * 20` slots. `packet_count = ceil(total_leds / 20)`.

## Hard-won quirks (read before coding against this)

1. **The packet index is 1-BASED.** Sending index 0 makes the firmware
   drop off the USB bus entirely. This cost us an evening.
2. **`FD 01` (software mode) makes the hub re-enumerate.** It disappears
   from USB and returns, possibly under a different hidraw node — and the
   *old* node can linger briefly, so a quick reopen can grab a corpse.
   Reopen and **ping with a status query until it answers** before
   streaming ("proof of life before paint").
3. **Colors latch.** The hub holds the last streamed frame indefinitely
   with zero refresh — event-driven control works fine. Power loss resets
   to the hub's built-in effects (a warm reboot does not: the hub is
   SATA-powered).
4. **The diodes have no gamma.** High-brightness mixed colors wash toward
   white; run saturated colors dark (true amber ≈ `200,110,10`, not
   `245,189,99`).
5. **Return-to-hardware-effects command: unknown.** `FD 00` is untried.
   A full power cycle restores the hub's own effects.
6. **The pip `hidapi` wheel may use the libusb backend**, which ignores
   hidraw permissions and fails to open the device. This tool avoids the
   problem by using `/dev/hidraw` directly — no dependencies.

## Credits

- **[glassknug](https://github.com/glassknug)** — desktop GUI, software effects, fan curves,
  system sensor integration, desktop launcher and Bazzite hardware testing.

- **u/idk_a_name_wtf** (r/ASIAHORSEGlobal) — original Wireshark capture
  of the passthrough packets, proving the hub could be spoken to at all.
- **maihcx** — [SignalRGB plugin](https://github.com/maihcx/AsiaHorse-Umbra-ARGB-Hub-SignalRGB-Plugin)
  documenting the full protocol.
- **ToeTagger** — found the sources, supplied the hardware, eyeballed
  every color; a guy in Arkansas who wouldn't stop digging. Linux port
  and quirk hunting with **Claude**.

MIT licensed. If you port this into OpenRGB proper, the protocol notes
above are yours — that's the point.
