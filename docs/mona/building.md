# Building this kernel for the Moto G Stylus 2025

[Mona documentation](README.md)

This guide builds the **base kernel `Image`** from the `mona-android16` checkout
with Parrot/Mona settings. It then shows how to replace the kernel in a copy
of the phone's matching stock `boot.img`. It does not rebuild vendor modules,
DTBOs, or OEM packages.

Shell comments label read-only **CHECK** commands and file-changing **SETUP**,
**CONFIGURE**, **BUILD**, **COPY**, **RECORD**, **REPACK**, and **VERIFY**
commands. **VERIFY** may write temporary files. Run section 6 in one shell
so its variables persist.

## 1. Record what is checked out

From this repository:

```sh
# CHECK: inspect the checkout; these commands do not change it.
git status --short --branch
git log -1 --format='%H%n%s'
git describe --tags --always
head -n 5 Makefile
```

At the time of writing, HEAD is commit
`db75a01fbc826a67bef15539bc3fbb2b8369a627` (Linux **6.6.118**) with no
tag at HEAD. Check the phone's exact firmware before using the result;
`android15-6.6` in `build.config.common` names the common kernel branch, not
the phone's Android version.

This checkout has the Mona config but lacks the vendor DT tree and full Bazel
workspace required for Motorola's distribution target. Section 3 builds only
`Image` and does not compile DTBs or external modules.

## 2. Get the matching GKI build environment

Use a **64-bit x86 Linux host**. The `android15-6.6-2026-01` kernel manifest
downloads the Clang toolchain and build tools used below. It requires network
access and several gigabytes of disk space.

On Debian/Ubuntu, install the host programs and headers first:

```sh
# SETUP: install host packages.
sudo apt update
sudo apt install git repo build-essential bc bison flex libelf-dev libssl-dev \
  libncurses-dev dwarves python3 curl ca-certificates rsync cpio lz4 zstd file unzip adb
# CHECK: confirm Repo is available.
repo version
```

`dwarves` supplies `pahole` for BTF. If `repo` is unavailable through your
package manager, use the [official Repo installation instructions](https://source.android.com/docs/setup/start/requirements#install-repo).

Create a separate AOSP kernel checkout beside this repository:

```sh
# SETUP: create and populate the separate GKI environment checkout.
cd /home/barwal01/misc/moto-workspace
mkdir -p android-kernel-env
cd android-kernel-env
repo init -u https://android.googlesource.com/kernel/manifest \
  -b common-android15-6.6-2026-01
repo sync -c -j4
```

Adjust the paths if your workspace is elsewhere. After `repo sync`, check the
required tools:

```sh
# CHECK: confirm required projects, headers, and compiler are present.
test -f build/kernel/build-tools/sysroot/usr/include/string.h
test -d prebuilts/kernel-build-tools/linux-x86/lib64
test -x prebuilts/clang/host/linux-x86/clang-r510928/bin/clang
prebuilts/clang/host/linux-x86/clang-r510928/bin/clang --version
# RECORD: write the fetched project revisions to resolved-manifest.xml.
repo manifest -r -o resolved-manifest.xml
```

`resolved-manifest.xml` records the fetched revisions. If the compiler check
fails, inspect `prebuilts/clang/host/linux-x86/` before continuing.

## 3. Build the Parrot/Mona base `Image` with extra driver flags

Motorola's `perf` config order is `gki_defconfig`, `vendor/parrot_perf.config`,
then a fragment containing `moto-parrot.config`, `moto-parrot-mona.config`,
and `moto-parrot-perf.config`. Motorola's published Parrot recipe concatenates
the three Motorola files before invoking Bazel. The commands below use that
order and resolve it with `olddefconfig`. Bazel may apply further build
settings, so this does not establish an identical final `.config`. If you
later obtain a complete Mona Bazel workspace, compare its `.config` with
`$KERNEL_OUT/.config`.

Edit `arch/arm64/configs/vendor/ext_config/moto-parrot-mona.config` in this
repository. Keep its existing settings and add built-in drivers as
`CONFIG_SYMBOL=y`, one per line. Drivers set to `=m` require separate module
builds and deployment. Review the edit with
`git diff -- arch/arm64/configs/vendor/ext_config/moto-parrot-mona.config`.

`CONFIG_VIDEO_V4L2` does not exist in this tree. Use `CONFIG_VIDEO_DEV=y` for
the V4L2 core and `CONFIG_USB_VIDEO_CLASS=y` for USB webcams.

From `android-kernel-env`, run:

```sh
# SETUP: define source, output, and toolchain paths for this shell.
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
# CHECK: fail early if required source and toolchain files are missing.
test -f "$KERNEL_SRC/arch/arm64/configs/gki_defconfig"
test -x "$CLANG_BIN/clang"
test -f "$HOST_SYSROOT/usr/include/string.h"
test -s "$MONA_CONFIG"
# SETUP: select host tools.
export PATH="$CLANG_BIN:$PWD/build/kernel/build-tools/path/linux-x86:$PATH"
export LD_LIBRARY_PATH="$HOST_TOOLS/lib64${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
# CHECK: confirm selected host tools resolve in PATH.
command -v clang
command -v ld.lld
command -v pahole
# SETUP: create build directories and point temporary files there.
mkdir -p "$KERNEL_OUT"
mkdir -p "$KERNEL_OUT/tmp"
export TMPDIR="$KERNEL_OUT/tmp" TEMP="$KERNEL_OUT/tmp" TMP="$KERNEL_OUT/tmp"
# CONFIGURE: merge the Parrot/Mona fragments and write the final .config.
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
# CHECK: verify that required Mona settings survived configuration.
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
# BUILD: compile the kernel and write arch/arm64/boot/Image under KERNEL_OUT.
make -C "$KERNEL_SRC" O="$KERNEL_OUT" ARCH=arm64 LLVM=1 \
  HOSTCFLAGS="$HOST_CFLAGS" HOSTLDFLAGS="$HOST_LDFLAGS" -j4 Image
# CHECK: identify the resulting file.
file "$KERNEL_OUT/arch/arm64/boot/Image"
```

The artifact is `$KERNEL_OUT/arch/arm64/boot/Image`. The verification loop
checks that each `CONFIG_...=` assignment in the Mona fragment survived
`olddefconfig`; inspect `.config` separately for any `# CONFIG_... is not set`
lines you edited. The separator `printf` preserves fragment boundaries even
when a source fragment lacks a final newline.

If the build fails, save the **first** error. Check disk space and confirm
both `make` commands received the same toolchain flags. Rerun the failing
target with `-j1 V=1` for detailed output. The host sysroot flags avoid a
`libbpf.c` compile error caused by newer host glibc headers; keep them in
both `make` commands.

## 4. Identify the phone and exact software later

When you have the phone, enable USB debugging and record:

```sh
# CHECK: read phone identity, firmware, kernel, and active slot.
adb devices
adb shell getprop ro.product.device
adb shell getprop ro.product.model
adb shell getprop ro.build.version.release
adb shell getprop ro.build.fingerprint
adb shell getprop ro.build.version.incremental
adb shell uname -a
adb shell getprop ro.boot.slot_suffix
```

Confirm the device is Mona and match its full firmware build to the source
release and stock boot image. Android and kernel version numbers alone are
insufficient; carrier and monthly releases may differ.

Motorola typically names source tags `MMI-<build ID>`. Query the exact build:

```sh
# CHECK: query the phone's build ID and remote tag list; no files are changed.
BUILD_ID="$(adb shell getprop ro.build.id | tr -d '\r')"
git ls-remote --tags https://github.com/MotorolaMobilityLLC/kernel-msm.git \
  "refs/tags/MMI-${BUILD_ID}"
```

If no tag is returned, do not assume a similar tag matches. Check the
[Motorola build instructions](https://github.com/MotorolaMobilityLLC/readme)
for that release as well.

## 5. Where Motorola's separate build tasks fit

[Motorola's Parrot recipe](https://github.com/MotorolaMobilityLLC/readme/blob/master/W1VT36H.22-20.txt)
uses `portov` as its example product and builds the kernel distribution,
device trees, Qualcomm modules, and Motorola modules in separate tasks. The
`make ... Image` path above selects Mona and performs only the base kernel
build. It uses the listed config fragments without requiring the vendor DT
tree or Bazel product files absent from this checkout.

Use vendor and Motorola modules from the **same stock firmware** as the boot
image. Kernel interface changes can break those modules even when `Image`
builds. A driver needing a new module, DTB/DTBO property, or firmware file
requires deployment beyond an `Image` replacement.

## 6. Replace only the kernel payload in a copy of the phone's boot image

Use the `Image` from section 3 only after matching the phone's firmware and
checking that the drivers need no other files. Repacking preserves the other
extracted boot components but changes image metadata and invalidates the OEM
AVB signature.

### Get `magiskboot` on the Linux host

Download an official [Magisk APK](https://github.com/topjohnwu/Magisk/releases)
to the Linux host and set `MAGISK_APK` below to its path. Run all section 6
blocks in the same shell:

```sh
# SETUP: create a host work directory and extract magiskboot from the APK.
set -euo pipefail
cd /home/barwal01/misc/moto-workspace
BOOT_WORK="$(mktemp -d "$PWD/mona-boot-work.XXXXXX")"
mkdir -p "$BOOT_WORK/tools"
cd "$BOOT_WORK"
MAGISK_APK="/absolute/path/to/Magisk.apk"
# CHECK: confirm the APK contains the expected host binary.
unzip -l "$MAGISK_APK" 'lib/x86_64/libmagiskboot.so'
# SETUP: write and make the host binary executable.
unzip -p "$MAGISK_APK" 'lib/x86_64/libmagiskboot.so' > "$BOOT_WORK/tools/magiskboot"
chmod 755 "$BOOT_WORK/tools/magiskboot"
MAGISKBOOT="$BOOT_WORK/tools/magiskboot"
# CHECK: confirm extraction produced a nonempty file.
test -s "$MAGISKBOOT"
```

If the APK lacks this x86-64 library, obtain an x86-64 `magiskboot` from an
official Magisk release/build.

### Pull the active-slot `boot` partition

Pull the active-slot **`boot`** partition. These commands require working
`su` on the phone. If the phone is not rooted, obtain `boot.img` from the
exact matching stock firmware package. Do not substitute `init_boot` or
`vendor_boot`.

```sh
# CHECK: identify the active slot and its boot block device.
adb devices
SLOT="$(adb shell getprop ro.boot.slot_suffix | tr -d '\r')"
case "$SLOT" in _a|_b) ;; *) echo "Unexpected slot: $SLOT" >&2; exit 1 ;; esac
adb shell su -c "ls -l /dev/block/by-name/boot${SLOT}"
# COPY: read the boot partition; write copies to phone Download and BOOT_WORK.
adb shell su -c "dd if=/dev/block/by-name/boot${SLOT} of=/sdcard/Download/boot${SLOT}-stock.img bs=4M"
adb pull "/sdcard/Download/boot${SLOT}-stock.img" "$BOOT_WORK/boot-stock.img"
# CHECK: confirm the copied file has Android boot image magic.
test "$(head -c 8 "$BOOT_WORK/boot-stock.img")" = 'ANDROID!'
# RECORD: write checksum, slot, and fingerprint files beside the copy.
sha256sum "$BOOT_WORK/boot-stock.img" > "$BOOT_WORK/boot-stock.img.sha256"
printf '%s\n' "$SLOT" > "$BOOT_WORK/active-slot.txt"
adb shell getprop ro.build.fingerprint > "$BOOT_WORK/phone-fingerprint.txt"
```

The `dd` command reads the boot partition and writes a copy to the phone's
Download directory; `adb pull` copies it to the host. If `su` or the boot
magic check fails, stop. Pull a fresh image after an OTA or slot change.

### Unpack, replace, and repack

Set `IMAGE` to the section 3 artifact. Keep the original extraction for
comparison; replace only `modified/kernel`.

```sh
# CHECK: confirm the built kernel Image exists.
IMAGE="/home/barwal01/misc/moto-workspace/android-kernel-env/out/mona-parrot-image/arch/arm64/boot/Image"
test -s "$IMAGE"
# REPACK: unpack stock boot, replace its kernel, and write boot-custom.img.
mkdir -p "$BOOT_WORK/original" "$BOOT_WORK/modified" "$BOOT_WORK/verified"
(cd "$BOOT_WORK/original" && "$MAGISKBOOT" unpack "$BOOT_WORK/boot-stock.img")
# CHECK: confirm stock boot yielded a kernel.
test -s "$BOOT_WORK/original/kernel"
# REPACK: copy all components, replace only kernel, and rebuild the image.
cp -a "$BOOT_WORK/original/." "$BOOT_WORK/modified/"
cp "$IMAGE" "$BOOT_WORK/modified/kernel"
(cd "$BOOT_WORK/modified" && env -u PATCHVBMETAFLAG "$MAGISKBOOT" repack \
  "$BOOT_WORK/boot-stock.img" "$BOOT_WORK/boot-custom.img")
# CHECK: confirm a repacked image was written.
test -s "$BOOT_WORK/boot-custom.img"
```

Do not use `unpack -n` or `repack -n`; they change compression handling.
Leaving `PATCHVBMETAFLAG` unset avoids requesting Magisk's vbmeta-disable
behavior.

### Verify the repacked payload before using it

The repacked kernel must equal `Image`; all other extracted components must
equal those from the stock image.

```sh
# VERIFY: unpack boot-custom.img into BOOT_WORK/verified for comparison.
(cd "$BOOT_WORK/verified" && "$MAGISKBOOT" unpack "$BOOT_WORK/boot-custom.img")
# CHECK: compare the repacked kernel with the built Image.
cmp "$IMAGE" "$BOOT_WORK/verified/kernel"
# VERIFY: write component lists, compare them, then compare each component.
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

If any comparison fails, do not use `boot-custom.img`. Inspect `magiskboot`
output for warnings. These checks cannot prove bootloader acceptance or
compatibility with installed vendor modules. Keep `boot-stock.img` for recovery.

## 7. Before any boot or flash attempt

Before testing `boot-custom.img`, verify the exact firmware, active slot,
partition size, bootloader unlock state, and device-specific flashing
procedure. Back up user data before unlocking the bootloader. Keep matching
stock images for recovery, and do not relock with custom images installed.

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
