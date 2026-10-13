# Building this kernel for the Moto G Stylus 2025

This guide describes the `mona-android16` checkout inspected on 2026-10-13.
Its main path builds the **base kernel `Image`** with the Parrot/Mona config
and your additional built-in driver settings. It does not rebuild Qualcomm
vendor modules, Motorola modules, DTBOs, or OEM packages. When you have the
phone, you can repack that `Image` into a copy of its matching stock `boot.img`.

## 1. Record what is checked out

From this repository:

```sh
git status --short --branch
git log -1 --format='%H%n%s'
git describe --tags --always
head -n 5 Makefile
```

At the time of writing, this is branch `mona-android16`, commit
`db75a01fbc826a67bef15539bc3fbb2b8369a627`, Linux **6.6.118**. There is
no Git tag pointing at HEAD, so treat the checkout as a branch revision, not an
exact Motorola firmware release tag. The `android15-6.6` value in
`build.config.common` names the Android Common Kernel branch; it does not by
itself tell you which Android version is installed on your phone.

The clues tying this tree to Mona are
`arch/arm64/configs/vendor/ext_config/moto-parrot-mona.config` and
`parrot.bzl`. `parrot` is the Qualcomm platform family; `mona` is the device
selection. This single repository is enough source for the direct `Image`
target in section 3, but not for Motorola's full Bazel distribution target:

- `build.config.moto` refers to generated `moto_fragment.config`, and
  `parrot.bzl` loads generated `moto_product.bzl`. Section 3 recreates the
  fragment merge for a direct `Image` build; section 5 explains the Bazel
  route.
- The Bazel rules also load `//build:msm_kernel_extensions.bzl`, and the legacy
  config expects sibling `msm-kernel`, `common`, and `external/dtc` projects.
- There is no `arch/arm64/boot/dts/vendor` tree here, so the Motorola device
  trees and overlays are absent.
- There is no `tools/bazel`, `build/build.sh`, or bundled Clang toolchain in
  this single repository.

The missing source-layout pieces matter to Motorola's distribution/package
target. The direct `Image` target does not compile DTBs or external modules.

## 2. Get the matching GKI build environment

These steps use a **64-bit x86 Linux host**. They fetch the Android kernel
manifest for the same GKI release family named in this checkout's commit
history: `android15-6.6-2026-01`. The manifest downloads Clang, build tools,
Kleaf, and their support projects. It is a kernel checkout, **not** the much
larger full Android platform checkout. It still needs ample disk space and
network access. The branch supplies a toolchain environment; it does not add
Motorola's missing files to this repository.

On Debian/Ubuntu, install the host programs and headers first:

```sh
sudo apt update
sudo apt install git repo build-essential bc bison flex libelf-dev libssl-dev \
  libncurses-dev dwarves python3 curl ca-certificates rsync cpio lz4 zstd file unzip
repo version
```

`dwarves` supplies `pahole`, needed because this tree's `gki_defconfig` enables
BTF. If your distribution has no `repo` package, use the [official Repo
installation instructions](https://source.android.com/docs/setup/start/requirements#install-repo).
The Android manifest supplies Clang, so there is no need to use your distro's
`clang` package for this path.

From the directory containing this repository (here called `moto-workspace`),
create a **separate** AOSP kernel checkout. Keep it separate so `repo sync`
cannot overwrite your current branch or this guide:

```sh
cd /home/barwal01/misc/moto-workspace
mkdir -p android-kernel-env
cd android-kernel-env
repo init -u https://android.googlesource.com/kernel/manifest \
  -b common-android15-6.6-2026-01
repo sync -c -j4
```

If you keep the repository elsewhere, change the `cd` and `KERNEL_SRC` path in
the next section. `repo sync` can take a while and transfer many gigabytes.
After it finishes, check that the expected projects and toolchain arrived:

```sh
test -x tools/bazel
test -d build/kernel
test -d common
test -f build/kernel/build-tools/sysroot/usr/include/string.h
test -d prebuilts/kernel-build-tools/linux-x86/lib64
test -x prebuilts/clang/host/linux-x86/clang-r510928/bin/clang
prebuilts/clang/host/linux-x86/clang-r510928/bin/clang --version
repo manifest -r -o resolved-manifest.xml
```

If the `clang-r510928` check fails, inspect
`prebuilts/clang/host/linux-x86/` and the manifest's resolved revision before
continuing. Do not silently switch compiler versions for a reproducible
device build. The `resolved-manifest.xml` file records exactly which projects
were fetched. A branch can move after this guide is written.

Optionally, verify that Google's downloaded GKI environment itself builds
before compiling this checkout:

```sh
tools/bazel run //common:kernel_aarch64_dist
```

This builds the manifest's **AOSP `common/` checkout**, not your Motorola
branch. Bazel prints the distribution directory when it finishes. The next
section compiles your checkout using the downloaded toolchain.

## 3. Build the Parrot/Mona base `Image` with extra driver flags

The repository's `build.config.msm.parrot` sources `build.config.msm.perf` and
`build.config.moto`. For the `perf` variant, that means: start with
`gki_defconfig`, add `vendor/parrot_perf.config`, then add the generated
Motorola fragment. Motorola's Parrot recipe creates that fragment by
concatenating `moto-parrot.config`, `moto-parrot-${TARGET_PRODUCT}.config`,
and `moto-parrot-perf.config`, in that order. For this phone,
`TARGET_PRODUCT=mona`. The first Moto file enables `CONFIG_MMI_DEVICE_DTBS`;
the Mona file enables `CONFIG_MONA_DTB` and Mona-specific drivers. The
`consolidate` variant adds other debug fragments and is not used below.

Edit this checkout's
`arch/arm64/configs/vendor/ext_config/moto-parrot-mona.config` directly.
Keep its existing settings and add your driver settings there, one per line.
For example, a new line has the form `CONFIG_YOUR_DRIVER=y`; replace
`YOUR_DRIVER` with a real Kconfig symbol. Save the file with a final newline.
For an `Image`-only change, the new driver and any needed dependencies must
resolve to `=y`. A setting of `=m` builds a separate `.ko` and will not add
that driver to `Image`; that path needs a module build and deployment. Review
your source change with `git diff -- arch/arm64/configs/vendor/ext_config/moto-parrot-mona.config`
from the kernel repository before building.

From `android-kernel-env`, set paths for the checkout and an out-of-tree
build directory. The Mona file is included in the Motorola fragment, so the
manual merge follows the repository's `perf` fragment order without any
additional config file. It does not need the missing vendor DT tree:

```sh
set -euo pipefail
KERNEL_SRC="$PWD/../moto-g-stylus-2025-kernel"
KERNEL_OUT="$PWD/out/mona-parrot-image"
CLANG_BIN="$PWD/prebuilts/clang/host/linux-x86/clang-r510928/bin"
HOST_SYSROOT="$PWD/build/kernel/build-tools/sysroot"
HOST_TOOLS="$PWD/prebuilts/kernel-build-tools/linux-x86"
HOST_CFLAGS="--sysroot=$HOST_SYSROOT -I$HOST_TOOLS/include"
HOST_LDFLAGS="--sysroot=$HOST_SYSROOT -L$HOST_TOOLS/lib64 -fuse-ld=lld --rtlib=compiler-rt"
MOTO_CONFIG_DIR="$KERNEL_SRC/arch/arm64/configs/vendor/ext_config"
MONA_CONFIG="$MOTO_CONFIG_DIR/moto-parrot-mona.config"
MOTO_FRAGMENT="$KERNEL_OUT/moto_fragment.config"
test -f "$KERNEL_SRC/arch/arm64/configs/gki_defconfig"
test -x "$CLANG_BIN/clang"
test -f "$HOST_SYSROOT/usr/include/string.h"
test -s "$MONA_CONFIG"
export PATH="$CLANG_BIN:$PWD/build/kernel/build-tools/path/linux-x86:$PATH"
export LD_LIBRARY_PATH="$HOST_TOOLS/lib64${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
command -v clang
command -v ld.lld
command -v pahole
mkdir -p "$KERNEL_OUT"
mkdir -p "$KERNEL_OUT/tmp"
export TMPDIR="$KERNEL_OUT/tmp" TEMP="$KERNEL_OUT/tmp" TMP="$KERNEL_OUT/tmp"
for fragment in \
  "$MOTO_CONFIG_DIR/moto-parrot.config" \
  "$MONA_CONFIG" \
  "$MOTO_CONFIG_DIR/moto-parrot-perf.config"; do
  cat "$fragment"
  printf '\n'
done > "$MOTO_FRAGMENT"
KCONFIG_CONFIG="$KERNEL_OUT/.config" \
  "$KERNEL_SRC/scripts/kconfig/merge_config.sh" -m -r -y \
  "$KERNEL_SRC/arch/arm64/configs/gki_defconfig" \
  "$KERNEL_SRC/arch/arm64/configs/vendor/parrot_perf.config" \
  "$MOTO_FRAGMENT"
make -C "$KERNEL_SRC" O="$KERNEL_OUT" ARCH=arm64 LLVM=1 \
  HOSTCFLAGS="$HOST_CFLAGS" HOSTLDFLAGS="$HOST_LDFLAGS" olddefconfig
grep -qx 'CONFIG_ARCH_PARROT=y' "$KERNEL_OUT/.config"
grep -qx 'CONFIG_MMI_DEVICE_DTBS=y' "$KERNEL_OUT/.config"
grep -qx 'CONFIG_MONA_DTB=y' "$KERNEL_OUT/.config"
grep -qx 'CONFIG_SCSI_SKHID=y' "$KERNEL_OUT/.config"
while IFS= read -r setting; do
  grep -Fqx "$setting" "$KERNEL_OUT/.config" || {
    echo "Mona setting did not survive olddefconfig: $setting" >&2
    exit 1
  }
done < <(grep -E '^CONFIG_[A-Za-z0-9_]+=' "$MONA_CONFIG")
make -C "$KERNEL_SRC" O="$KERNEL_OUT" ARCH=arm64 LLVM=1 \
  HOSTCFLAGS="$HOST_CFLAGS" HOSTLDFLAGS="$HOST_LDFLAGS" -j4 Image
file "$KERNEL_OUT/arch/arm64/boot/Image"
```

The result is `$KERNEL_OUT/arch/arm64/boot/Image`. Increase `-j4` only if
your machine has enough RAM. The `O=` path keeps generated files outside
the source checkout. For a clean rebuild, remove only the `KERNEL_OUT`
directory after confirming its value.

The loop checks that every `CONFIG_...=` assignment in the edited Mona file
survived `olddefconfig`, including your new settings. Inspect the final
`.config` separately for any settings expressed as `# CONFIG_... is not set`.
An unrecognized symbol can appear in the merged text but disappear when
`olddefconfig` resolves it. This build creates a real Parrot/Mona kernel
payload, but its compatibility still depends on the phone's exact firmware
and existing vendor modules. Section 4 covers that match before repacking.

The separator `printf` calls preserve fragment boundaries. The original Mona
file in this checkout lacks a final newline; plain concatenation would join
its last setting to the next fragment's first comment and lose that setting.

If compilation fails, save the **first** error, not only the last lines of
`make` output. Verify `clang --version`, `pahole --version`, free disk space,
and that the two `make` commands used the same `KERNEL_OUT`, `ARCH`, `LLVM`,
`HOSTCFLAGS`, and `HOSTLDFLAGS` values. For a detailed single-job command,
rerun the failing target with `-j1 V=1`.

The host sysroot and linker flags above follow
[Motorola's Parrot recipe](https://github.com/MotorolaMobilityLLC/readme/blob/master/W1VT36H.22-20.txt).
They matter on hosts with glibc 2.43: its headers make `strchr` preserve
`const`, exposing an older `tools/lib/bpf/libbpf.c` declaration as a build
error under `-Werror`. Using the pinned sysroot avoids that host-header
mismatch. If you see `libbpf.c:11255` fail, check that both `make` commands
received the `HOSTCFLAGS` and `HOSTLDFLAGS` shown here. The direct `Image`
build was verified with the source line unmodified and these flags.

## 4. Identify the phone and exact software later

When you have the phone, enable USB debugging and record:

```sh
adb devices
adb shell getprop ro.product.device
adb shell getprop ro.product.model
adb shell getprop ro.build.version.release
adb shell getprop ro.build.fingerprint
adb shell getprop ro.build.version.incremental
adb shell uname -a
adb shell getprop ro.boot.slot_suffix
```

Look for Mona (`mona` or a market-specific variant), the exact Motorola build
number/fingerprint, and the running kernel version. Compare the full build
number with the Motorola source release or manifest that supplied the missing
projects. **Android 16 and kernel 6.6 alone are insufficient matches.** A
different carrier, region, or monthly update may use different kernel,
modules, or boot images. Keep a copy of the stock firmware for that exact
build before experimenting with boot images.

Motorola typically names source tags `MMI-<build ID>`. Check for the exact
phone build before choosing a release (this only queries remote refs):

```sh
BUILD_ID="$(adb shell getprop ro.build.id | tr -d '\r')"
git ls-remote --tags https://github.com/MotorolaMobilityLLC/kernel-msm.git \
  "refs/tags/MMI-${BUILD_ID}"
```

An empty result means that this exact tag was not found by the query. Do not
replace it with a similar-looking carrier or security-update tag. Also check
the corresponding [Motorola build instructions repository](https://github.com/MotorolaMobilityLLC/readme)
for the release's source layout and dependencies.

## 5. Where Motorola's separate build tasks fit

[Motorola's published Parrot recipe](https://github.com/MotorolaMobilityLLC/readme/blob/master/W1VT36H.22-20.txt)
first prepares the product config and runs a Bazel kernel distribution target.
After that it prepares the module build environment, compiles vendor device
trees, builds Qualcomm DLKMs, and builds Motorola DLKMs as separate steps.
The later device tree and external module steps are **not required** to run
the `make ... Image` command in section 3.

The published recipe's `//msm-kernel:parrot_perf_dist` target is a broader
distribution target than `make Image`. In this checkout its Bazel rules load
vendor device-tree definitions that are absent from this single repository,
so that target is not the route used here. The direct build in section 3
reproduces its base defconfig/fragment order without invoking the Bazel
packaging, DTB, or module targets. Its generated `moto_fragment.config` lives
in the output directory; `moto_product.bzl` is needed only for the Bazel
product selection and is not needed for the direct `Image` build.

Keep the installed vendor and Motorola modules from the **same stock
firmware** as the boot image. An extra `=y` driver is compiled into the new
kernel; an extra `=m` driver needs a separate `.ko` and module deployment.
Changing kernel interfaces or configuration that existing modules rely on
can break those stock modules even when the new `Image` compiles. If your
intended driver needs new DTB/DTBO properties, firmware files, or a module,
an `Image` replacement alone cannot supply those dependencies.

## 6. Replace only the kernel payload in a copy of the phone's boot image

Use the `Image` from section 3 **after** confirming the phone's exact firmware
matches the kernel source and that your new built-in driver needs no other
deployed files. A rebuild of vendor or OEM packages is not part of this path.
The goal is to preserve the boot image's other extracted components while
substituting the kernel payload. Repacking necessarily updates fields such as
the kernel size/checksum, and an OEM AVB signature cannot remain valid after
the kernel bytes change. This does not promise a bootable or byte-identical
image outside the kernel.

### Get `magiskboot` on the Linux host

Download the official [Magisk APK release](https://github.com/topjohnwu/Magisk/releases)
to the host. The APK is a ZIP archive; on an x86-64 Linux host, extract its
`magiskboot` executable. The host-package command in section 2 includes `unzip`.
Use a fresh work directory **outside** both source checkouts. Run the rest of
section 6 in the same shell; `set -euo pipefail` stops the sequence if a
command fails:

```sh
set -euo pipefail
cd /home/barwal01/misc/moto-workspace
BOOT_WORK="$(mktemp -d "$PWD/mona-boot-work.XXXXXX")"
mkdir -p "$BOOT_WORK/tools"
cd "$BOOT_WORK"
MAGISK_APK="/absolute/path/to/Magisk.apk"
unzip -l "$MAGISK_APK" 'lib/x86_64/libmagiskboot.so'
unzip -p "$MAGISK_APK" 'lib/x86_64/libmagiskboot.so' > "$BOOT_WORK/tools/magiskboot"
chmod 755 "$BOOT_WORK/tools/magiskboot"
MAGISKBOOT="$BOOT_WORK/tools/magiskboot"
test -s "$MAGISKBOOT"
```

If that APK does not contain the x86-64 library, obtain a Linux x86-64
`magiskboot` from an official Magisk release/build. Do not use an ARM Android
binary on an x86-64 host. Keep the APK version with your build notes so a
later repack uses the same tool.

### Pull the active-slot `boot` partition

The kernel is in **`boot`**, even on devices whose Magisk installation uses
`init_boot` for its ramdisk. Do not substitute `init_boot` or `vendor_boot`.
The commands below require working root (`su`) on the phone; ordinary `adb
pull` cannot read a protected boot block device. If the phone is not rooted,
obtain `boot.img` from the **exact matching stock firmware package** instead,
and verify that it matches the currently installed build and slot.

```sh
adb devices
SLOT="$(adb shell getprop ro.boot.slot_suffix | tr -d '\r')"
case "$SLOT" in _a|_b) ;; *) echo "Unexpected slot: $SLOT" >&2; exit 1 ;; esac
adb shell su -c "ls -l /dev/block/by-name/boot${SLOT}"
adb shell su -c "dd if=/dev/block/by-name/boot${SLOT} of=/sdcard/Download/boot${SLOT}-stock.img bs=4M"
adb pull "/sdcard/Download/boot${SLOT}-stock.img" "$BOOT_WORK/boot-stock.img"
test "$(head -c 8 "$BOOT_WORK/boot-stock.img")" = 'ANDROID!'
sha256sum "$BOOT_WORK/boot-stock.img" > "$BOOT_WORK/boot-stock.img.sha256"
printf '%s\n' "$SLOT" > "$BOOT_WORK/active-slot.txt"
adb shell getprop ro.build.fingerprint > "$BOOT_WORK/phone-fingerprint.txt"
```

These commands **read** the phone's boot partition and write a copy to its
Download directory and your host. They do not write the partition. Save the
stock image, SHA-256 file, active slot, and full firmware fingerprint together.
If `su` is denied or the boot magic test fails, stop and use the exact stock
firmware image; do not repack a partial or unknown file. If an OTA changes the
active slot or firmware, pull a new boot image.

### Unpack, replace, and repack

Set `IMAGE` to the kernel binary from section 3. `magiskboot`
normally decompresses the extracted kernel and recompresses it to the
original format during repack. Keep the original extraction for comparison;
only replace `modified/kernel`.

```sh
IMAGE="/home/barwal01/misc/moto-workspace/android-kernel-env/out/mona-parrot-image/arch/arm64/boot/Image"
test -s "$IMAGE"
mkdir -p "$BOOT_WORK/original" "$BOOT_WORK/modified" "$BOOT_WORK/verified"
(cd "$BOOT_WORK/original" && "$MAGISKBOOT" unpack "$BOOT_WORK/boot-stock.img")
test -s "$BOOT_WORK/original/kernel"
cp -a "$BOOT_WORK/original/." "$BOOT_WORK/modified/"
cp "$IMAGE" "$BOOT_WORK/modified/kernel"
(cd "$BOOT_WORK/modified" && env -u PATCHVBMETAFLAG "$MAGISKBOOT" repack \
  "$BOOT_WORK/boot-stock.img" "$BOOT_WORK/boot-custom.img")
test -s "$BOOT_WORK/boot-custom.img"
```

The work directory used for repacking contains only the component files
extracted by `magiskboot`; the original image, logs, and tool live outside it.
Do not use `unpack -n` or `repack -n` in this recipe: those options change the
automatic compression handling. Leaving `PATCHVBMETAFLAG` unset avoids
requesting Magisk's optional vbmeta-disable behavior.

### Verify the repacked payload before using it

Unpack the new image independently. Its `kernel` must equal your built
`Image`, and every other extracted component must equal the corresponding
component from the original image. These checks also catch a repack that
silently kept the stock kernel.

```sh
(cd "$BOOT_WORK/verified" && "$MAGISKBOOT" unpack "$BOOT_WORK/boot-custom.img")
cmp "$IMAGE" "$BOOT_WORK/verified/kernel"
(cd "$BOOT_WORK/original" && find . -maxdepth 1 -type f ! -name kernel -printf '%f\n' | sort) \
  > "$BOOT_WORK/original-components.txt"
(cd "$BOOT_WORK/verified" && find . -maxdepth 1 -type f ! -name kernel -printf '%f\n' | sort) \
  > "$BOOT_WORK/verified-components.txt"
diff -u "$BOOT_WORK/original-components.txt" "$BOOT_WORK/verified-components.txt"
while IFS= read -r part; do
  cmp "$BOOT_WORK/original/$part" "$BOOT_WORK/verified/$part" || exit 1
done < "$BOOT_WORK/original-components.txt"
sha256sum "$BOOT_WORK/boot-custom.img"
```

If any comparison fails, do not use `boot-custom.img`. Also inspect the
`magiskboot` output for compression, size, or signing warnings. The checks
compare extracted components; they cannot prove the bootloader will accept
the altered AVB/GKI signature or that the new kernel remains compatible with
the installed vendor modules. Keep `boot-stock.img` for recovery. If your
phone's current boot image was already patched, this workflow preserves those
extracted changes instead of removing them.

## 7. Before any boot or flash attempt

A raw `Image` is **not** a `boot.img`. Before testing `boot-custom.img`, verify
the exact firmware match, the active slot, the boot partition size, the
bootloader unlock state, and the device-specific flashing procedure. Do not
flash `init_boot`, `vendor_boot`, DTBO, or module partitions as part of this
kernel-only replacement. Unlocking a bootloader normally wipes user data;
arrange a personal-data backup first. Keep the matching stock images available
for recovery, and do not relock the bootloader with custom images installed.

## References

- [Android: Build kernels](https://source.android.com/docs/setup/build/building-kernels)
- [Android: AOSP host setup and Repo installation](https://source.android.com/docs/setup/start/requirements)
- [Android kernel manifest: `common-android15-6.6-2026-01`](https://android.googlesource.com/kernel/manifest/+/refs/heads/common-android15-6.6-2026-01)
- [Kernel documentation: LLVM builds](https://docs.kernel.org/kbuild/llvm.html)
- [Android: Kernel branches and build systems](https://source.android.com/docs/setup/reference/bazel-support)
- [Android: Generic boot partition](https://source.android.com/docs/core/architecture/partitions/generic-boot)
- [Magisk: `magiskboot` tools and repack behavior](https://topjohnwu.github.io/Magisk/tools.html)
- [Magisk: official releases](https://github.com/topjohnwu/Magisk/releases)
- [Motorola: published Parrot build recipe (example product)](https://github.com/MotorolaMobilityLLC/readme/blob/master/W1VT36H.22-20.txt)
- [Android: Lock and unlock the bootloader](https://source.android.com/docs/core/architecture/bootloader/locking_unlocking)
