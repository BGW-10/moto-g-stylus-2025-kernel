# Manual fake SOC on Mona

[Mona documentation](README.md)

`arch/arm64/configs/vendor/ext_config/moto-parrot-mona.config` enables
`CONFIG_QTI_PMIC_GLINK_CLIENT_DEBUG=y`. This activates the existing capacity
override and `power_supply_changed()` notification in
`drivers/power/supply/qti_battery_charger.c`. No driver changes or boot-time
writes are needed. The driver still initializes `fake_soc` to `-EINVAL`
(`-22`), so normal reporting is the default.

## Configuration and build

Follow the configuration merge in [the build guide](building.md):
`gki_defconfig`, `vendor/parrot_perf.config`, then the concatenation of
`moto-parrot.config`, `moto-parrot-mona.config`, and `moto-parrot-perf.config`,
followed by `olddefconfig`. The guide's Mona assignment check includes the
new setting. Check the actual output `.config` before building:

```text
CONFIG_RPMSG=y
CONFIG_DEBUG_FS=y
CONFIG_QTI_PMIC_GLINK=m
CONFIG_QTI_PMIC_GLINK_CLIENT_DEBUG=y
CONFIG_QTI_BATTERY_CHARGER=m
```

The debug option is a boolean depending on `QTI_PMIC_GLINK && DEBUG_FS`;
the existing modular GLINK setting satisfies it. It selects no additional
options. Keep the existing USB serial settings, including
`CONFIG_USB_SERIAL_CP210X=y`.

**An Image-only replacement is insufficient.** The charger is configured as
a module: this compile-time change must reach the loaded
`qti_battery_charger.ko`. Rebuild and package the corresponding modules with
the matching kernel configuration/toolchain and firmware-compatible module
set. The Image-only build guide does not perform that packaging. Check module
ABI, signatures, and the device's module loading location before deployment;
do not assume a stock charger module will gain the override from a new Image.
The complete Motorola Bazel/vendor distribution workspace is not present in
this repository, so its final config and module packaging need separate
validation if that workflow is used.

## Additional debug controls

This flag covers more than fake SOC. Kconfig explicitly warns against enabling
it on production devices because writes can change physical parameters and
cause instability. In this tree it also enables:

- `battery_debug/votables/<name>/override` (debugfs, mode `0600`) in
  `qti_battery_debug.c`: writes send charging-vote overrides to firmware.
- `altmode/send_pan_en` and `altmode/send_pan_ack` (debugfs, mode `0200`) in
  `altmode-glink.c`: writes trigger USB alternate-mode protocol messages.

Both associated drivers are already configured as modules. The controls
appear when rebuilt drivers probe successfully and debugfs is accessible.
Enabling the flag does not itself write overrides, but these privileged
interfaces expand what a process with sufficient access can change. Leave
them unused; preserve SELinux and existing debugfs access restrictions.
`battery_charger/block_tx` is already gated by `DEBUG_FS`, independently of
this option. `QTI_PMIC_GLINK_DEBUG` is also a separate existing option.

The configuration approach is retained for this custom host because it uses
the existing implementation without source changes. A fake-SOC-only source
option would require a separate patch if the extra debug controls are
unacceptable.

## Verify after deployment

Before setting an override, confirm `fake_soc` reads `-22`. Record battery
`capacity`, `voltage_now`, `current_now`, and `temp` from
`/sys/class/power_supply/battery/`. Then, from PowerShell:

```powershell
adb shell "su -c 'echo 80 > /sys/class/qcom-battery/fake_soc'"
adb shell "su -c 'cat /sys/class/qcom-battery/fake_soc'"
adb shell "su -c 'cat /sys/class/power_supply/battery/capacity'"
adb shell dumpsys battery
adb shell "su -c 'cat /sys/class/power_supply/battery/voltage_now /sys/class/power_supply/battery/current_now /sys/class/power_supply/battery/temp'"
```

Expect `fake_soc` and kernel capacity to read `80`, and Android's level to
update to `80`. Voltage, current, and temperature retain their existing
measurement paths; they may naturally vary between samples. Android's health
stack must consume this power supply for the Android level to follow it.
If only `fake_soc` changes, verify the loaded charger module was rebuilt.
If kernel capacity changes but Android's level does not, inspect the health
HAL's source and any active Android battery simulation.

Restore normal reporting and repeat the capacity/Android checks:

```powershell
adb shell "su -c 'echo -22 > /sys/class/qcom-battery/fake_soc'"
```

Only values from `0` through `100` activate the override; `-22` restores the
default sentinel. A reboot should again leave the override inactive. No
automatic activation is supplied.

This changes reported Linux capacity, not firmware fuel-gauge state. Thermal,
charging, and low-voltage protection code is unchanged. Firmware shutdowns
can still occur, and other consumers of reported capacity can change behavior.
The capacity getter still reads firmware before applying the override, so
fake SOC is not a bypass for GLINK read failures.

The live phone also loads `bcl_soc`, which computes battery depletion as
`100 - battery/capacity`. A working override of 80 should therefore report 20
in the `socd` thermal zone and affect depletion-based mitigation. Separate
voltage/current BCL protection remains active. See
[the appliance investigation](appliance.md) for live observations,
XL4015 power-delivery considerations, and service/performance recommendations.
