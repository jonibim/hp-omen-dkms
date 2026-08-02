# Battery charge control

`charge_behaviour` / `charge_control_end_threshold` support depending on DMI.

Current support:
- `8C2F` (HP Victus 15-fb2xxx).

## Sysfs interface

Standard `power_supply` ABI on `/sys/class/power_supply/BAT0/`, registered
via `devm_battery_hook_register()`:

```bash
# Raw mechanism: auto / inhibit-charge / force-discharge, no policy attached.
cat  /sys/class/power_supply/BAT0/charge_behaviour
echo inhibit-charge | sudo tee /sys/class/power_supply/BAT0/charge_behaviour

# Policy convenience: enforced in software (polls every HP_WMI_CHARGE_POLL_SECS,
# currently 60s) since the EC has no native percentage register, the same
# approach other in-tree drivers use for threshold-less hardware. Charging
# stops at the threshold but only resumes HP_WMI_CHARGE_HYSTERESIS percentage
# points below it (currently 3), so it doesn't rapidly toggle on/off from
# normal capacity fluctuation right at the edge. This is the property KDE's
# Battery Charge Limit slider and TLP's
# START_CHARGE_THRESH_BATx/STOP_CHARGE_THRESH_BATx already read/write.
echo 80 | sudo tee /sys/class/power_supply/BAT0/charge_control_end_threshold
```

- `charge_behaviour`: `auto` / `inhibit-charge` / `force-discharge`. Pure
  mechanism, no driver-side policy. Writing this directly cancels any active
  threshold (an explicit override should not be fought by the poller
  `HP_WMI_CHARGE_POLL_SECS` later).
- `charge_control_end_threshold`: `20`-`100`, or `100` to disable. Writing
  it re-arms threshold enforcement.

**NOTE:** neither KDE
PowerDevil's native charge-limit UI nor TLP's threshold config can drive a
`charge_behaviour`-only device to a percentage: both are built around
`charge_control_start/end_threshold` specifically (KDE bug
[#449997](https://bugs.kde.org/show_bug.cgi?id=449997)).

**Persistence across reboots** can be done via a module parameter, since
`charge_control_end_threshold` itself resets to disabled on every module
reload (in-memory only):

```bash
# /etc/modprobe.d/hp-wmi.conf
options hp_wmi battery_charge_threshold=80
```

`charge_control_end_threshold` and this `battery_charge_threshold` module
parameter are two different *kinds* of kernel interface. `charge_control_end_threshold` is a standard
`power_supply` property with a `set_property` that can be written. Whereas the `battery_charge_threshold` is a module parameter exposed under `/sys/module/hp_wmi/parameters/` mainly so its value
can be inspected. This driver only ever reads it **once**, at the exact
moment it loads (boot, or a manual `modprobe`).

## The mechanism

### Background

WMI is just a way for the OS to call a function living in the laptop's
firmware. HP has ~80 custom features reachable this way (battery charge
control, fan speed, thermal profile, keyboard backlight, ...), and rather
than expose a separately-named WMI method per feature, HP funnels most of
them through one shared entry point, with an extra number in every call
saying which feature is actually being requested. That number is the
"commandtype"; `0x2B` is one specific value of it, used here for battery
charge control. `hp_wmi_perform_query(active_charge_params->commandtype, ...)`
in `hp-wmi.c` is what packages that number up and sends it over WMI.

On the firmware side, this number arrives at a dispatch table, a plain list
of checks (visible directly in the decompiled DSDT) that maps each number
to the function that should handle it:

```
If ((CMDT == 0x01))  { Local2 = GDST() }   // command 0x01 -> run GDST
If ((CMDT == 0x04))  { Local2 = GDKS() }   // command 0x04 -> run GDKS
If ((CMDT == 0x2B))  { Local2 = GBCO() }   // command 0x2B -> run GBCO
```

That table may not be standard and may depending on the laptop model's firmware. A different model may get a different mapping table.

### `GBCO`/`SBCO`

WMI command `0x2B` maps to `GBCO`/`SBCO` in the DSDT, HP's own "Battery
Health Manager" mechanism. `SBCO` writes a mode byte to the EC's `BDVO`
field; `GBCO` reads back a status byte. Both were found by decompiling DSDT.

> Note: ACPI names are capped at 4 characters (ACPI namespace
format).

`GBCO`/`SBCO` may stand for "Get/Set Battery Charge
Option". `BDVO` (and `MBDC`, used by the separate
`0x1F`/`GBCC`/`SBCC` mechanism) are EC field names with unknown abbreviation/internal names.

An upstream RFC patch (`platform/x86: hp-wmi: Add charge behaviour support`,
submitted 2026-05-31, targeting DMI board `85C6`) independently arrived at
the same commands.

## Mode survey

Every write-side mode `0x00`-`0x07` was probed live, one at a time, with
immediate recovery via mode `0x00` afterward:

| Write mode | Accepted? | `GBCO` readback | Effect | Used? |
|---|---|---|---|---|
| `0x00` | yes | `0x00` | Normal/full charging | **Yes, `auto`** |
| `0x01` | yes | `0x03` | Same practical "not charging" effect as `0x05`, via a different internal EC path | No, redundant, unverified at other capacity levels |
| `0x02` | yes | `0x02` | **Actively discharges the battery (~19-35 W observed) while AC is connected**, rather than merely pausing charge | **Yes, `force-discharge`** (exposed as its own explicit behaviour, never used by the threshold poller) |
| `0x03` | **no** | n/a | Fails outright with an ACPI interpreter error (`AE_AML_OPERAND_TYPE`) on this firmware | No, incompatible? |
| `0x04` | yes | `0x00` | Indistinguishable from writing `0x00` directly | No, no apparent benefit |
| `0x05` | yes | `0x04` | Stops charging cleanly | **Yes, `inhibit-charge`** |
| `0x06` | untested | n/a | Also sets `EC0.SHPM`, name suggestive of ship-mode power cutoff | No, risk of unexpected shutdown/unusual recovery, left untested |
| `0x07` | untested | n/a | Also sets `EC0.STOE`, name suggestive of storage-mode power cutoff | No, same reason as `0x06` |

### Additional behaviour

**`GBCO`'s readback is not an echo of the last mode written.** It derives
its returned byte on the fly from several *other* EC registers
(`R820`, `R760`, `R830`, `R4C0`, `R610`), and takes on different non-zero
values while charging is progressing through ordinary phases
(`0x01`-`0x03` observed with no charge control active at all, mid-charge).
The *only* readback value confirmed to mean
"charging is actually inhibited" is `0x04`.

In testing, treating any non-zero readback as "already
inhibited" made the driver mistake a normal charging
phase (`0x01`, observed while actively charging at 95% with no limit
configured) for the desired inhibited state and
let the battery charge straight past the configured threshold. Fixed by
checking specifically for `0x04` on every poll, rather than trusting a
cached flag: the EC can drift into or out of the inhibited state
independently of this driver (e.g. an AC replug was observed to reset it to
`0x00` on its own), so the live value has to be re-checked every time, not
just on state transitions.

**Enforcement is skipped while on battery** as it can cause weird behaviours. While unplugged with capacity above the configured threshold, it may
keep re-sending the inhibit command every poll. This leaves
the EC's own `_BST` status computation stuck reporting `fully-charged` for a battery that is being discharging. HP's firmware appears to derive
charge/discharge status partly from the same state `SBCO` controls. This is fixed
by checking `power_supply_is_system_supplied()`: on battery, any lingering
inhibit is cleared once and the EC is otherwise left untouched entirely.

## Adding another board

Parameters (WMI commandtype, and the four mode/readback byte values) live in
`struct battery_charge_params`, referenced per board via `driver_data` in
`hp_wmi_charge_control_quirks[]`, the same shape as
`struct thermal_profile_params`/`victus_s_thermal_profile_boards`. There is
currently only one instance (`battery_charge_gbco_params`), since every
board confirmed so far (`8C2F` here, `85C6` upstream) agrees on both the
command and every value.

However, it needs further testing on other models: command `0x2B` may be a
per-model WMI dispatch table entry and a different
board generation could use a different mechanism entirely rather than just
different values. For example, this same board's own DSDT (`8C2F`) also
has an unrelated `0x1F`/`GBCC`/`SBCC` mechanism, writing a different EC
field, `MBDC`, with only 3 modes.

Adding a new board means repeating the process that found this one:

1. Decompile that board's DSDT (`acpidump` + `iasl -d`) and confirm `GBCO`/
   `SBCO` exist, and read `SBCO`'s body to see what it actually writes and
   with what values (static analysis, no live risk).
2. If it matches (or diverges, note the new values), read-only `GBCO` on
   real hardware first.
3. A single controlled `auto` -> `inhibit` -> `auto` round trip, watching
   `power_now`/status throughout.
4. Add the board's DMI name to `hp_wmi_charge_control_quirks`, pointing at
   `battery_charge_gbco_params` if the values matched, or a new
   `battery_charge_params` instance if they didn't.

## Verification commands

The actual commands used to test this live, before and after it was wired
into the driver.

Step 2 (raw `GBCO`/`SBCO` calls via `acpi_call`, independent of the driver,
used to confirm the mechanism before writing any driver code):

```bash
sudo modprobe acpi_call

# Read current state (GBCO takes no arguments)
echo "\_SB.WMID.GBCO" > /proc/acpi/call
cat /proc/acpi/call
# [0x0, 0x4, {0x00, 0xff, 0x00, 0x00}]   <- mode byte 0x00 = not inhibited

# Write mode 0x05 (Arg0=0, Arg1=5, Arg2=0, Arg3=0) to inhibit charging
echo "\_SB.WMID.SBCO BUFQ{0x00, 0x05, 0x00, 0x00}" > /proc/acpi/call
cat /proc/acpi/call
# [0x0, 0x0, {0x00, 0x00, 0x00, 0x00}]   <- status 0x0 = accepted

# Confirm the EC actually changed state
echo "\_SB.WMID.GBCO" > /proc/acpi/call
cat /proc/acpi/call
# [0x0, 0x4, {0x04, 0xff, 0x00, 0x00}]   <- mode byte 0x04 = confirmed inhibited

# Cross-check against the battery's real reported behaviour
cat /sys/class/power_supply/BAT0/status      # Not charging
cat /sys/class/power_supply/BAT0/power_now   # 0

# Revert (Arg1=0, mode 0x00 = auto/full)
echo "\_SB.WMID.SBCO BUFQ{0x00, 0x00, 0x00, 0x00}" > /proc/acpi/call
```

Step 3 (once the driver implemented `charge_behaviour`, verifying it through
the real sysfs interface and cross-checking against the same raw `GBCO`
call, to prove the two agree):

```bash
echo inhibit-charge | sudo tee /sys/class/power_supply/BAT0/charge_behaviour
cat /sys/class/power_supply/BAT0/charge_behaviour
# auto [inhibit-charge] force-discharge

# Bypass the driver and read the EC's own state directly, independently
echo "\_SB.WMID.GBCO" > /proc/acpi/call
cat /proc/acpi/call
# [0x0, 0x4, {0x04, 0xff, 0x00, 0x00}]   <- matches: driver and EC agree
```
