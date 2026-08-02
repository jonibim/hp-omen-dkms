# omen-dkms

DKMS package providing an enhanced `hp-wmi` kernel module for HP OMEN and Victus
gaming laptops on Linux. This driver extends the upstream Linux `hp-wmi` module
with board support, sysfs interfaces, and fan-control reliability fixes needed
for recent OMEN Slim and related models.

The module replaces the in-tree `hp_wmi` driver at load time via DKMS and exposes
standard interfaces used by tools such as [OmenCore](https://github.com/theantipopau/omencore).

## Supported hardware

This fork is developed and tested primarily on:

| Board ID | Example model | Notes |
|----------|---------------|-------|
| `8D40` | OMEN Slim 16-an0xxx / 16-an0015tx | Victus-S thermal path, four-zone RGB, improved fan AUTO handoff |
| `8E35` | HP OMEN 16 | Four-zone RGB, GPU thermal mode probing |
| `8C2F` | HP Victus 15-fb2xxx (AMD) | Victus-S thermal path; EC readback confirmed at offset `0x95` by live testing. Four-zone sysfs nodes exist but this board has no 4-zone RGB keyboard, so they're non-functional. |

Other boards already supported by the imported upstream driver may work unchanged.
Adding a board ID requires confirming the correct thermal profile table and
firmware behavior for that SKU.

## Changes from upstream

This repository is based on the Linux kernel `hp-wmi` driver (imported at
`cef99c0`) with the following additions and fixes (`789dfa1` and later):

### Board support

- Added DMI board `8D40` to `victus_s_thermal_profile_boards` with
  `omen_v1_no_ec_thermal_params`, enabling the Victus-S platform profile and
  hwmon path on OMEN Slim 16 hardware.
- Added `8E35` to `omen_thermal_profile_boards` and
  `omen_timed_thermal_profile_boards` where missing from the imported base.
- Added DMI board `8C2F` (HP Victus 15-fb2xxx AMD) to
  `victus_s_thermal_profile_boards` with a new `victus_s_amd_thermal_params`.

### Battery charge control (standard power_supply ABI)

Added HP's "Battery Health Manager" mechanism (WMI command `0x2B`,
`GBCO`/`SBCO`), gated to DMI board `8C2F`. Exposed through the standard
Linux `power_supply` ABI on the real `BAT0` device via `devm_battery_hook_register()`: `charge_behaviour`
(`auto`/`inhibit-charge`/`force-discharge`) and
`charge_control_end_threshold` (percentage cap).

```bash
echo 80 | sudo tee /sys/class/power_supply/BAT0/charge_control_end_threshold
```

### Four-zone keyboard backlight (sysfs)

New WMI command definitions and sysfs attributes under
`/sys/devices/platform/hp-wmi/`:

| Attribute | Access | Description |
|-----------|--------|-------------|
| `fourzone_color` | read/write | 24-character hex string (4 zones x 6 hex digits RGB) |
| `fourzone_brightness` | read/write | Global brightness (0-255) |
| `fourzone_animation` | read/write | Firmware animation mode index |

Positive WMI return codes from the EC are treated as errors and propagated to
userspace as I/O errors.

**Physical zone layout** (left to right on a full-size OMEN keyboard):

| Zone | Location |
|------|----------|
| Zone 3 | Left edge (Esc / Tab / modifier column) |
| Zone 4 | WASD cluster |
| Zone 2 | Middle section (G row through center) |
| Zone 1 | Right section and numpad (including middle Enter) |

The sysfs string order is Zone 1, Zone 2, Zone 3, Zone 4 as defined by the
firmware protocol.

Example: set all zones to OMEN blue (`00BFFF`):

```bash
echo "00BFFF00BFFF00BFFF00BFFF" | sudo tee /sys/devices/platform/hp-wmi/fourzone_color
```

### Fan control reliability

- Added `hp_wmi_fan_speed_max_get()` and `hp_wmi_fan_speed_max_set_verify()` with
  retry logic so max-mode transitions are confirmed by the EC.
- Added `hp_wmi_omen_exit_userdefined_mode()` to reliably leave experimental
  fan-stop / user-defined PWM mode on boards that do not clear state on a
  single `max_set(0)`.
- Added `hp_wmi_omen_fan_speed_reset()` for Victus-S manual fan baseline.
- `hp_wmi_apply_fan_settings()` now accepts `prev_mode` and only runs the full
  exit-pulse sequence when transitioning from `PWM_MODE_MANUAL` to
  `PWM_MODE_AUTO`, avoiding unnecessary fan bursts on other transitions.
- Replaced `cancel_delayed_work_sync()` with `cancel_delayed_work()` when
  holding `priv->lock`, preventing a deadlock with the keep-alive worker.

### GPU thermal modes

- Replaced a hardcoded board check with `omen_has_gpu_thermal_modes()`, which
  probes Victus-S CTGP/PPAB WMI support at runtime so compatible firmware is
  used without maintaining a board-name allowlist.

### Build dependencies

- Added `#include <linux/delay.h>` and `#include <linux/hex.h>` for the above.

For a line-level view of fork changes against the import baseline:

```bash
git diff cef99c0..HEAD -- hp-wmi.c
```

## Requirements

- Linux with kernel headers installed (`/lib/modules/$(uname -r)/build`)
- DKMS (`dkms` package)
- Build tools: `gcc`, `make`
- Root access to load kernel modules

Supported distributions: any distro with DKMS (Arch, CachyOS, Fedora, Ubuntu,
Debian, Pika OS, etc.).

### Distribution packages

**Debian / Ubuntu / Pika OS:**

```bash
sudo apt update
sudo apt install dkms build-essential linux-headers-$(uname -r)
```

**Arch / CachyOS:**

```bash
sudo pacman -S dkms linux-headers
```

## Installation

Clone this repository and install with DKMS. The package version is defined in
`dkms.conf` (currently **1.1.0** — use that version in all `dkms` commands):

```bash
git clone https://github.com/saikiran2001-v2/hp-omen-dkms.git omen-dkms
cd omen-dkms

sudo dkms add .
sudo dkms build hp-wmi/1.1.0
sudo dkms install hp-wmi/1.1.0
```

Reload the module:

```bash
sudo modprobe -r hp_wmi
sudo modprobe hp_wmi
```

Confirm the DKMS build is active (not the stock in-tree module):

```bash
modinfo -n hp_wmi
# expected: /lib/modules/$(uname -r)/updates/dkms/hp-wmi.ko
```

Verify platform profile registration:

```bash
cat /sys/class/platform-profile/*/choices
cat /sys/class/platform-profile/*/profile
```

Verify keyboard sysfs nodes:

```bash
ls /sys/devices/platform/hp-wmi/fourzone_*
```

Verify hwmon fan interface:

```bash
ls /sys/devices/platform/hp-wmi/hwmon/hwmon*/pwm1_enable
```

## OmenCore and keyboard lighting

The DKMS module exposes `fourzone_color`, `fourzone_brightness`, and
`fourzone_animation` under `/sys/devices/platform/hp-wmi/`. The driver is only
half the story: those sysfs files are created as **root-owned and not writable
by your desktop user**. OmenCore (GUI and CLI) writes to them directly.

On CachyOS you may have had the OmenCore udev rule installed already. On a
fresh Debian/Pika OS install it is usually missing, so lighting appears broken
even when DKMS is installed correctly.

### Install the OmenCore udev rule (recommended for GUI)

From your [OmenCore](https://github.com/theantipopau/omencore) checkout:

Install the helper script and udev rule (the rule calls the script on module load):

```bash
sudo mkdir -p /usr/local/lib/omencore
sudo cp scripts/init-fourzone-keyboard.sh /usr/local/lib/omencore/
sudo chmod +x /usr/local/lib/omencore/init-fourzone-keyboard.sh
sudo cp scripts/99-omencore-hp-wmi.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=platform --action=change
```

If you build OmenCore with `build.sh`, it installs both files automatically.

Verify permissions (should show `666` after the rule runs):

```bash
ls -l /sys/devices/platform/hp-wmi/fourzone_*
```

### Hardware brightness gate (`fourzone_brightness`)

Four-zone keyboards use **two** sysfs controls:

| Node | Purpose |
|------|---------|
| `fourzone_color` | Per-zone RGB (24 hex chars: 4 zones × RRGGBB) |
| `fourzone_brightness` | Hardware brightness gate (0–255) |

Firmware often boots with `fourzone_brightness = 0` on a fresh distro install
(Fedora, Debian, Pika OS, etc.). Color writes succeed and OmenCore may report
success while the keyboard stays physically off. This is **not** a udev failure —
check brightness before blaming permissions.

```bash
cat /sys/devices/platform/hp-wmi/fourzone_brightness
cat /sys/devices/platform/hp-wmi/fourzone_color
```

Fix manually:

```bash
echo 255 | sudo tee /sys/devices/platform/hp-wmi/fourzone_brightness
omencore-cli keyboard --color 00BFFF
```

Or via OmenCore:

```bash
omencore-cli keyboard --brightness 100
omencore-cli keyboard --color 00BFFF
```

`init-fourzone-keyboard.sh` (installed with the udev rule) raises
`fourzone_brightness` from `0` to `255` when the `hp-wmi` platform device
appears. Set `OMENCORE_SKIP_BRIGHTNESS_INIT=1` to chmod only and preserve an
intentional off state.

### Quick sysfs test

```bash
# read current colors (24 hex chars)
cat /sys/devices/platform/hp-wmi/fourzone_color

# set all zones to OMEN blue (needs root OR the udev rule above)
echo "00BFFF00BFFF00BFFF00BFFF" | sudo tee /sys/devices/platform/hp-wmi/fourzone_color
```

### OmenCore without udev rules

Run CLI commands with sudo, or launch the GUI elevated:

```bash
sudo omencore-cli keyboard --color 00BFFF
pkexec omencore-gui
```

## Updating

After pulling new changes:

```bash
sudo dkms remove hp-wmi/1.1.0 --all
sudo dkms add .
sudo dkms install hp-wmi/1.1.0
sudo modprobe -r hp_wmi && sudo modprobe hp_wmi
```

## Uninstallation

```bash
sudo modprobe -r hp_wmi
sudo dkms remove hp-wmi/1.1.0 --all
sudo modprobe hp_wmi   # loads the stock in-tree module, if present
```

## Troubleshooting

### Keyboard lighting does nothing in OmenCore

1. **Confirm DKMS module is loaded** (not stock `hp_wmi`):
   ```bash
   modinfo -n hp_wmi
   ls /sys/devices/platform/hp-wmi/fourzone_*
   ```
2. **Check `fourzone_brightness`** — if it is `0`, the hardware gate is off even
   when colors are set. See [Hardware brightness gate](#hardware-brightness-gate-fourzone_brightness).
   ```bash
   cat /sys/devices/platform/hp-wmi/fourzone_brightness
   echo 255 | sudo tee /sys/devices/platform/hp-wmi/fourzone_brightness
   ```
3. **Check sysfs permissions** — if files are `-rw-r--r-- root root`, install
   the udev rule and `init-fourzone-keyboard.sh` in
   [OmenCore and keyboard lighting](#omencore-and-keyboard-lighting)
   or use `sudo` / `pkexec`.
4. **Reload the module** after DKMS install:
   ```bash
   sudo modprobe -r hp_wmi && sudo modprobe hp_wmi
   ```
5. **Test outside OmenCore**:
   ```bash
   echo 255 | sudo tee /sys/devices/platform/hp-wmi/fourzone_brightness
   echo "FF0000FF0000FF0000FF0000" | sudo tee /sys/devices/platform/hp-wmi/fourzone_color
   ```
   If sysfs works but OmenCore GUI does not, re-check the udev rule. If color
   writes work but the keyboard stays dark, brightness is almost certainly `0`.

### DKMS build fails

Ensure headers match the running kernel:

```bash
uname -r
dpkg -l "linux-headers-$(uname -r)"   # Debian/Pika OS
pacman -Q "linux-headers-$(uname -r)" # Arch/CachyOS
rpm -q "kernel-devel-$(uname -r)"     # Fedora
```

## Platform profile values

On Victus-S boards such as `8D40`, available profiles typically include:

- `low-power`
- `balanced`
- `performance`

Set via:

```bash
echo balanced | sudo tee /sys/class/platform-profile/*/profile
```

## Fan control (hwmon)

Fan mode is exposed through the hwmon `pwm1_enable` attribute:

| Value | Mode |
|-------|------|
| `0` | Max (immediate full speed) |
| `1` | Manual / user-defined (fan-stop capable on supported boards) |
| `2` | Auto (BIOS/EC managed) |

Paths vary by kernel version; locate with:

```bash
find /sys/devices/platform/hp-wmi -name pwm1_enable
```

## License and attribution

This project is licensed under the **GNU General Public License v2.0** (or
later). See [LICENSE](LICENSE) for the full text.

Upstream `hp-wmi` is Copyright (C) Matthew Garrett, Anssi Hannula, and other
Linux kernel contributors. Modifications in this repository are Copyright (C)
2026 saikiran.

You may modify and redistribute this code under the GPL. If you redistribute
it, you must preserve copyright notices and document your changes. Do not
present this work or substantial portions of it as your own without attribution.
See [NOTICE](NOTICE) for details.

## Contributing

Contributions are welcome. Please:

1. Describe the board ID and hardware tested.
2. Keep changes focused and documented in commit messages.
3. Preserve existing copyright headers in `hp-wmi.c`.
4. Do not remove attribution to upstream or fork authors.

## Disclaimer

This is an unofficial community driver. It is not affiliated with or endorsed
by HP. Loading an out-of-tree kernel module may affect system stability,
thermal behavior, and warranty coverage depending on your jurisdiction and
hardware. Test on your own system and keep backups of working configurations.
