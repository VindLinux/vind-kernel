# Vind Kernel — Troubleshooting

Known build- and boot-time issues for the Vind kernel, one per section. Check here before treating something as a new bug — most of what shows up building these profiles has already been diagnosed once.

Each entry follows the same shape: what you'll see, why it happens, and what to do about it.

---

## `.zst`-compressed firmware missing from initramfs (Intel/AMD)

**Applies to:** [Intel](VIND.md#intel), [AMD](VIND.md#amd), [Generic](VIND.md#generic) (any profile with `i915` or `amdgpu` enabled).

**Symptom:** GPU/DRM init failure at boot — `i915` or `amdgpu` reporting a wedged GPU, no `/dev/dri/renderD128`.

**Cause:** These profiles enable the kernel's built-in compressed firmware support (`CONFIG_FW_LOADER_COMPRESS=y`, `CONFIG_FW_LOADER_COMPRESS_ZSTD=y`), which lets a driver request a plain firmware name (e.g. `i915/adlp_guc_70.bin`) and transparently get the `.zst`-compressed variant that `linux-firmware` actually ships today, decompressed in-kernel. The failure is almost never the kernel failing to read the file — it's **the `.zst` file never making it into the initramfs in the first place**. This is a documented upstream `dracut` limitation: firmware install detection historically didn't account for the `.zst` suffix, silently omitting the file. A fix landed upstream, but wildcard-named firmware entries can still be affected depending on the `dracut` version in use. The same detection gap applies to `amdgpu`'s firmware under `/usr/lib/firmware/amdgpu/`, for the same reason — it's a `dracut`/initramfs-generation issue, not driver-specific.

**Fix:**

1. Confirm the firmware exists on disk: `ls /usr/lib/firmware/i915/ | grep adlp_guc` (or `amdgpu` for AMD).
2. Check whether it's actually in the current initramfs: `lsinitrd /boot/initramfs-<version>.img | grep i915` (or `amdgpu`).
3. If missing, force it in via a `dracut.conf.d` override:

   ```
   # /etc/dracut.conf.d/i915-firmware.conf
   install_items+=" /usr/lib/firmware/i915/* "
   ```

   ```
   # /etc/dracut.conf.d/amdgpu-firmware.conf
   install_items+=" /usr/lib/firmware/amdgpu/* "
   ```

4. Regenerate the initramfs (`dracut --force ...`) and re-check with `lsinitrd`.

**Do not** work around this by manually decompressing firmware under `/lib/firmware` and rebuilding — it only fixes one machine, gets undone by the next `linux-firmware` update, and throws away the disk-space benefit the compressed format exists for. Treat the `dracut.conf.d` override as a standard part of any Intel or AMD Vind install, not a one-off fix — and add both overrides pre-emptively on [Generic](VIND.md#generic), since either GPU driver may end up in use depending on the target machine.

---

## `CONFIG_WERROR` vs amdgpu DC/DML stack frame size

**Applies to:** [AMD](VIND.md#amd), [Generic](VIND.md#generic) (any profile with `CONFIG_DRM_AMDGPU`).

**Symptom:** build failure in `drivers/gpu/drm/amd/display/dc/dml/`:

```
error: stack frame size (2088) exceeds limit (2048) in
  'dml31_ModeSupportAndSystemConfigurationFull' [-Werror,-Wframe-larger-than]
```

**Cause:** The AMD display driver's mode-calculation code has functions with large stack frames. Its own Makefile already raises the frame-size warning threshold for these specific files to `-Wframe-larger-than=2048` (above the kernel-wide default), because upstream knows this code runs close to the limit. Depending on the compiler version in use, some functions can still exceed even that raised threshold — a known, compiler-sensitive characteristic of upstream AMD's code, not a Vind regression. With `CONFIG_WERROR=y`, the warning becomes a hard build failure.

**Fix:** Disable `CONFIG_WERROR` for any profile building `amdgpu`. This is why it's already unset (`# CONFIG_WERROR is not set`) in the Generic profile's defconfig; if you hit this on the AMD vendor-specific profile (which still ships `CONFIG_WERROR=y`), apply the same change there. Downgrading the compiler to one that happens to fit this code under 2048 bytes works but is fragile and toolchain-specific — not a lasting fix. Patching the DC Makefile to raise the threshold further is an option too, but goes against keeping this upstream code unmodified (see [Philosophy](VIND.md#philosophy)); disabling `WERROR` is the lower-cost choice.

---

## `CONFIG_MODULES` silently unset: `=m` gets promoted to `=y`

**Applies to:** any profile fragment where `=m` is used and the resulting `.config` isn't checked afterward. Bit [Generic](VIND.md#generic) hardest historically, since it's the only profile that ever used `=m` at all.

**Symptom:** a defconfig fragment marks drivers `=m` (e.g. `DRM_AMDGPU=m`, `SATA_AHCI=m`), but the resulting build has no `.ko` files, `modules_install` only installs `modules.builtin`/`modules.builtin.modinfo`, and `dracut` fails with something like:

```
dracut[W]: /usr/lib/modules/<version>/modules.dep is missing. Did you run depmod?
```

You may also see the driver fail to build as if it were builtin (e.g. hitting the amdgpu DC/DML `-Werror` issue above) even though the fragment says `=m`.

**Cause:** `CONFIG_MODULES` has no `default y` in Kconfig — if nothing sets it, it's `n`. Generating a profile's `.config` from a bare `make <defconfig>` (as opposed to a `merge_config.sh` run over a base that already has `CONFIG_MODULES=y`) resolves purely from Kconfig defaults. When `CONFIG_MODULES` ends up `n`, any tristate symbol marked `=m` in the fragment gets **silently promoted to `=y`** by the Kconfig resolver, since a kernel without module support can't actually produce a `.ko` — the only way to "have" the driver is to build it in. The build succeeds, and the resulting kernel is fully monolithic whether that was the intent or not.

**Fix:** Decide deliberately rather than let this happen by accident:
- **Going monolithic (current default for all four profiles):** don't mark anything `=m` in the fragment — mark it `=y` directly, and set `# CONFIG_MODULES is not set` explicitly so the intent is documented, not incidental. This is the current state of all four Vind profiles.
- **Wanting real `.ko` modules instead:** add `CONFIG_MODULES=y` explicitly to the fragment before anything marked `=m`. After regenerating, confirm with `grep CONFIG_MODULES .config` that it actually landed as `=y` — then `=m` symbols will build as loadable modules and `modules_install`/`dracut` will have something to work with. Note this reopens the initramfs requirement for any `=m` driver needed to find/mount root (see [VIND.md](VIND.md#initramfs-optional)).

---

## `SCSI_DH` is bool, not tristate

**Symptom:** `olddefconfig`/defconfig generation prints:

```
arch/x86/configs/<defconfig>:NN:warning: symbol value 'm' invalid for SCSI_DH
```

**Cause:** `CONFIG_SCSI_DH` is a `bool` symbol in this kernel version, not `tristate` — it only accepts `y`/`n`. Setting `=m` in a fragment is accepted at the file level but rejected during Kconfig resolution, with the value silently forced to `n` (or occasionally `y`, depending on what selects it) instead of what was written.

**Fix:** Use `CONFIG_SCSI_DH=y` (never `=m`) whenever this symbol appears in a fragment. More generally: any `symbol value '<x>' invalid for <SYMBOL>` warning during defconfig generation means the fragment's type assumption for that symbol (tristate vs. bool) is wrong for this kernel version — fix the value in the fragment rather than ignoring the warning, since the resolver's silent substitution may not match what you actually wanted.

---

## GPU firmware on a monolithic build (Generic)

**Applies to:** [Generic](VIND.md#generic) only — the vendor-specific profiles don't mix GPU vendors, so this tradeoff doesn't come up for them the same way.

**Symptom:** kernel boots fine (console via `DRM_SIMPLEDRM`/`DRM_EFIDRM`), but `amdgpu`/`i915` come up without hardware acceleration.

**Cause:** `DRM_AMDGPU` and `DRM_I915` are builtin in this profile, so they initialize during early boot — before any initramfs stage and before the real root filesystem is mounted. Both need firmware blobs (GuC/DMC for Intel, PSP/VCN for AMD) to bring up acceleration, and `request_firmware()` has nowhere to read them from that early unless one of the options below is taken.

**This is a decision, not a bug** — pick based on whether the target needs GPU acceleration available before userspace comes up:

- **Option A — install `linux-firmware`, rely on late reprobe.** Simplest to maintain; firmware stays upgradeable independently of the kernel via the distro's package manager. The driver initializes without firmware at early boot, and a udev/systemd rule rebinds it once the real root (and `/lib/firmware`) is available. Right choice for a general-purpose distro where acceleration doesn't need to be up before userspace starts. If you additionally choose to stage firmware via a minimal initramfs rather than waiting for late reprobe, the [`.zst`-compressed firmware](#zst-compressed-firmware-missing-from-initramfs-intelamd) caveat above applies again.
- **Option B — bake firmware into the kernel with `CONFIG_EXTRA_FIRMWARE`.** Set `CONFIG_EXTRA_FIRMWARE_DIR` to a local checkout of `linux-firmware`, and list the exact files needed under `CONFIG_EXTRA_FIRMWARE` (chip-specific — GuC/HuC versions vary by Intel generation, PSP/VCN by AMD generation). Makes acceleration available immediately at boot with no userspace dependency, at the cost of a significantly larger `vmlinux`/`bzImage` and firmware that only updates when the kernel is rebuilt.

VM targets are unaffected either way — `DRM_VIRTIO_GPU`/`DRM_BOCHS`/`DRM_CIRRUS_QEMU` require no firmware.

---

## Version string mismatch between `VIND.md` and the actual source tree

**Symptom:**

```
realpath: /lib/modules/7.2.0-vind-generic-minimal: No such file or directory
dracut[F]: Cannot find module directory
```

...even though the build just succeeded, and `make modules_install` shows a different version:

```
INSTALL /lib/modules/7.2.5-vind-generic-minimal/modules.builtin
```

**Cause:** [VIND.md](VIND.md#version) documents a Linux version number, but that number is only updated by hand and can drift from whatever's actually checked out. Naming a `bzImage`/initramfs from the documented version instead of the real one produces a mismatch: `dracut`'s second argument must exactly match the directory `make modules_install` created under `/lib/modules/`, which is derived from the kernel's actual `Makefile` (`VERSION.PATCHLEVEL.SUBLEVEL`) plus `CONFIG_LOCALVERSION` — not from this document.

**Fix:** Before naming anything, check the real version:

```sh
make kernelversion
```

or just look at what `modules_install` printed. Use that value (not the one in `VIND.md`) for the `bzImage` filename, the `dracut` command, and the initramfs filename. If they genuinely differ, update `VIND.md`'s Version section too, so the next person doesn't hit the same mismatch.
