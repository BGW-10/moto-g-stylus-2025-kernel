# Mona kernel and appliance documentation

[Repository home](../../README.md)

Guides for the Motorola Moto G Stylus 5G (2025), codename `mona`, using the
`mona-android16` kernel as the base for an externally powered Android/Linux host.
The `mona-appliance` branch collects these changes for testing and refinement
before merging back into `mona-android16`.

## Guides

| Guide | Contents |
| --- | --- |
| [Appliance investigation](appliance.md) | Start here: live phone observations, XL4015 supply requirements, performance, cooling, and service lifecycle. |
| [USB-C power protection](usb-power.md) | Reverse-current-blocking hardware candidates, charging-disable software, and electrical qualification. |
| [Manual fake SOC](fake-soc.md) | Kernel configuration, additional debug controls, module deployment, and manual enable/restore commands. |
| [Building the kernel](building.md) | Toolchain setup, configuration merging, base Image build, and boot-image preparation. |

Shell examples that use repository-relative paths assume the repository root
as the working directory, unless the guide says otherwise.

## Validation and remaining work

As of the 2026-10-16 investigation:

- The persistent [Mona configuration fragment](../../arch/arm64/configs/vendor/ext_config/moto-parrot-mona.config)
  enables `CONFIG_QTI_PMIC_GLINK_CLIENT_DEBUG=y`. Configuration merging and
  `olddefconfig` retained it; the before/after comparison changed only this
  option. Existing USB serial/CP210X settings were preserved.
- The three affected driver objects compiled successfully. A complete kernel
  and vendor module distribution has not been built or deployed for this change.
  Fake SOC requires the rebuilt charger module; replacing only `Image` is
  insufficient. See [configuration and build](fake-soc.md#configuration-and-build).
- The phone investigation used read-only root ADB commands. Battery overrides,
  charging controls, performance changes, and Magisk changes were not applied.
- Hardware recommendations and software charging inhibition still require
  measurement and testing. See the [deployment sequence](appliance.md#deployment-and-qualification-sequence)
  and [USB power qualification](usb-power.md#qualification-before-counting-this-as-two-working-layers).

Fake SOC remains inactive until explicitly enabled. These changes retain
thermal, voltage, charging, and SELinux protections. A capacity override does
not change the firmware fuel gauge or guarantee uninterrupted operation.
