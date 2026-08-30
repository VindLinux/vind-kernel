# Vind Kernel

This repository contains the Linux kernel used by **Vind Linux**.

Vind maintains a small set of kernel-specific changes on top of the upstream Linux kernel, keeping the upstream source as the base whenever possible.

## Version

- **Linux:** 7.2.0
- **Architecture:** x86_64

## Configuration

Vind provides minimal x86_64 kernel configurations, split by CPU vendor. See the [Intel](#intel) and [AMD](#amd) sections below for details specific to each.

Hardware-specific driver configuration beyond what each vendor baseline enables is the user's responsibility. Before building the kernel for a physical machine, review the hardware and enable any additional required drivers in `make menuconfig`, `make nconfig`, or another kernel configuration interface. Pay particular attention to:

- Storage controllers (e.g. `SATA_AHCI`, `BLK_DEV_NVME`).
- Wi-Fi support and device drivers (e.g. `WLAN`, `CFG80211`, `MAC80211`, and chipset drivers).
- Input (`SERIO_I8042` + `KEYBOARD_ATKBD` for internal laptop keyboards, `USB_SUPPORT` + `HID_SUPPORT` + `USB_HID` for USB peripherals).
- Graphics driver matching the actual GPU.

The minimal configurations should be treated as a baseline rather than a universal hardware configuration.

---

## Intel

- **Status:** available
- **Config file:** `arch/x86/configs/vind_x86_64_intel_minimal_defconfig`
- **Generate with:** `make vind_x86_64_intel_minimal_defconfig`

### Coverage

The Intel minimal configuration targets modern Intel platforms (Alder Lake and newer tested) and includes:

- Graphics: `CONFIG_DRM_I915` (integrated Intel graphics), plus `DRM_EFIDRM`/`DRM_SIMPLEDRM` as early boot fallback.
- Networking: `CONFIG_E1000E` and `CONFIG_IGC` (wired), `CONFIG_IWLWIFI` (wireless).
- Power management: `CONFIG_X86_INTEL_PSTATE`, `CONFIG_INTEL_IDLE`, `CONFIG_ENERGY_MODEL`, governor default `schedutil`.
- I/O: `CONFIG_VMD` (Intel Volume Management Device — required on many laptops where NVMe is hidden behind it even without RAID configured) paired with `CONFIG_INTEL_IOMMU` for VT-d isolation/DMA protection.
- Platform: `CONFIG_I2C_I801` (commonly needed for touchpad/sensors on Intel laptops).
- CPU: microcode loading (`CONFIG_MICROCODE=y`) and Intel-specific CPU support (`CONFIG_CPU_SUP_INTEL`) are covered by kernel defaults and don't need explicit tuning.
- Crypto: `CONFIG_CRYPTO_AES_NI_INTEL` for hardware-accelerated AES.

### Known issue: `.zst`-compressed firmware missing from initramfs

The Intel configuration enables the kernel's built-in compressed firmware support (`CONFIG_FW_LOADER_COMPRESS=y`, `CONFIG_FW_LOADER_COMPRESS_ZSTD=y`). This lets drivers request a plain firmware name (e.g. `i915/adlp_guc_70.bin`) and have the kernel transparently fall back to a `.zst`-compressed variant and decompress it in-kernel — this is how `linux-firmware` ships firmware today, and no manual decompression should ever be required on a correctly built system.

If a driver fails to load its firmware at boot (commonly surfaced as GPU/DRM init failures, e.g. `i915` reporting a wedged GPU and no `/dev/dri/renderD128`), this is almost always because **the required `.zst` firmware file was never copied into the initramfs**, not because the kernel can't read it. This is a documented upstream `dracut` bug/limitation: firmware install detection has historically not accounted for the `.zst` suffix, silently omitting the file from the generated image. A fix landed upstream ("Fix firmware loading paths, support .zst-suffixed firmware"), but wildcard-named firmware entries can still be affected depending on the `dracut` version in use.

Do **not** work around this by manually decompressing firmware files under `/lib/firmware` and forcing a rebuild — this only fixes the symptom on one machine, is undone by the next `linux-firmware` update, and discards the disk-space benefit the compressed format exists for.

Instead:

1. Confirm the firmware exists on disk: `ls /usr/lib/firmware/i915/ | grep adlp_guc`.
2. Check whether it's actually in the current initramfs: `lsinitrd /boot/initramfs-<version>.img | grep i915`.
3. If missing, force it in via `/etc/dracut.conf.d/i915-firmware.conf`:

   ```
   install_items+=" /usr/lib/firmware/i915/* "
   ```

4. Regenerate the initramfs (`dracut --force ...`) and re-check with `lsinitrd`.

This keeps firmware shipped compressed (as intended) and survives future `linux-firmware` updates without per-machine intervention. Since this affects any Intel platform using GuC/DMC firmware (most Gen9+ integrated graphics), this `dracut.conf.d` override should be treated as a standard part of an Intel Vind install, not a one-off fix.

---

## AMD

- **Status:** available
- **Config file:** `arch/x86/configs/vind_x86_64_amd_minimal_defconfig`
- **Generate with:** `make vind_x86_64_amd_minimal_defconfig`

### Coverage

The AMD minimal configuration mirrors the Intel baseline where the hardware overlaps, with AMD-specific swaps:

- Graphics: `CONFIG_DRM_AMDGPU` instead of `CONFIG_DRM_I915`, plus `DRM_EFIDRM`/`DRM_SIMPLEDRM` as early boot fallback.
- Networking: `CONFIG_R8169` (Realtek, very common on AMD boards) alongside `CONFIG_E1000E`/`CONFIG_IGC` — many AMD boards (especially higher-end AM4/AM5) ship an Intel-vendor NIC despite the AMD CPU/chipset, so both are kept rather than assuming AMD implies Realtek. Wireless via `CONFIG_IWLWIFI`.
- Power management: `CONFIG_X86_AMD_PSTATE` instead of `CONFIG_X86_INTEL_PSTATE`. No `INTEL_IDLE` equivalent is needed — AMD relies on ACPI-based idle by default.
- I/O: `CONFIG_AMD_IOMMU` instead of `CONFIG_INTEL_IOMMU`. There is no `VMD` equivalent — AMD platforms don't hide NVMe controllers behind a bridging device the way some Intel laptops do, so `VMD` is intentionally absent here.
- Platform: `CONFIG_I2C_PIIX4` instead of `CONFIG_I2C_I801` (the AMD SMBus controller is a different chip entirely, not just a renamed Intel one). `CONFIG_SENSORS_K10TEMP` for native CPU temperature reporting.
- CPU: microcode loading (`CONFIG_MICROCODE=y`) is vendor-agnostic and covered by kernel defaults, same as Intel.
- Crypto: `CONFIG_CRYPTO_AES_NI_INTEL` is kept despite the name — AES-NI is an x86 instruction set extension present on both Intel and AMD CPUs, and the driver works on either.

### Anticipated firmware caveat

`amdgpu` also ships large firmware blobs under `/usr/lib/firmware/amdgpu/`, commonly `.zst`-compressed by `linux-firmware` for the same disk-space reasons as Intel's `i915` firmware. The same `dracut` `.zst` detection issue documented in the [Intel](#intel) section is expected to apply here too, since it's a `dracut`/initramfs-generation issue rather than anything driver-specific. Until confirmed against real AMD graphics hardware, treat this pre-emptively: add a matching `/etc/dracut.conf.d/amdgpu-firmware.conf` override (`install_items+=" /usr/lib/firmware/amdgpu/* "`) rather than waiting to hit the same wedged-GPU symptom the Intel section describes.

---

## VM

- **Status:** available
- **Config file:** `arch/x86/configs/vind_x86_64_vm_minimal_defconfig`
- **Generate with:** `make vind_x86_64_vm_minimal_defconfig`

### Coverage

The VM minimal configuration is designed around QEMU/KVM-style guests, prioritizing paravirtualized devices while retaining a small set of emulated-hardware fallbacks:

* **Virtualization:** `CONFIG_HYPERVISOR_GUEST`, `CONFIG_PARAVIRT`, `CONFIG_PARAVIRT_SPINLOCKS`, and `CONFIG_KVM_GUEST` enable the kernel's guest-specific optimizations when running under a hypervisor. These provide things such as paravirtualized spinlocks and optimized clock handling, reducing virtualization overhead instead of making the guest behave entirely like physical hardware.
* **Boot:** `CONFIG_PVH` provides PVH boot support, used by some Xen and cloud environments where the guest is entered directly by the hypervisor rather than through a traditional BIOS/UEFI path. Its compatibility value is high relative to its cost, so it remains enabled as a safety net.
* **VirtIO:** `CONFIG_VIRTIO_MENU`, `CONFIG_VIRTIO_PCI`, `CONFIG_VIRTIO_BLK`, `CONFIG_VIRTIO_NET`, `CONFIG_VIRTIO_BALLOON`, `CONFIG_VIRTIO_INPUT`, `CONFIG_VIRTIO_MMIO`, and `CONFIG_HW_RANDOM_VIRTIO` cover the main VirtIO device stack. VirtIO is the preferred paravirtualized interface for QEMU/KVM guests, providing efficient disk, networking, input, memory-ballooning, and host-provided entropy without relying on fully emulated hardware.
* **Emulated storage/network:** `CONFIG_E1000` and `CONFIG_ATA_PIIX` provide compatibility with classic emulated QEMU hardware. They act as fallbacks for machine types or VM configurations that do not expose VirtIO devices, keeping basic networking and storage functional at the cost of higher emulation overhead.
* **Graphics:** `CONFIG_DRM_VIRTIO_GPU` provides the paravirtualized GPU used by QEMU-based desktops, including setups using VirGL for accelerated rendering. `CONFIG_DRM_BOCHS` and `CONFIG_DRM_CIRRUS_QEMU` provide fallback support for traditional emulated VGA devices when VirtIO-GPU is unavailable.
* **Host<->guest filesystem:** `CONFIG_NET_9P`, `CONFIG_NET_9P_VIRTIO`, and `CONFIG_9P_FS` enable 9P over VirtIO, allowing QEMU's `virtfs` mechanism to expose host directories directly inside the guest. This is useful for shared folders without requiring a network filesystem.
* **USB:** `CONFIG_USB_UHCI_HCD` provides USB 1.1 UHCI controller support, covering older or simpler QEMU machine configurations where USB input devices are presented through an emulated UHCI controller.
* **Entropy:** `CONFIG_HW_RANDOM` is enabled as the parent facility for `CONFIG_HW_RANDOM_VIRTIO`. In a VM without a physical hardware RNG, VirtIO can expose entropy supplied by the host; without an available RNG source, early userspace or services that require entropy may have to wait for the guest's own entropy pool to initialize.

### Anticipated firmware caveat

The VM configuration does not require physical GPU firmware such as `amdgpu` or `i915` firmware when using QEMU's standard virtual graphics devices. `CONFIG_DRM_VIRTIO_GPU` provides the guest-side driver for VirtIO-GPU, while `CONFIG_DRM_BOCHS` and `CONFIG_DRM_CIRRUS_QEMU` cover common emulated VGA fallbacks.

No dedicated `/etc/dracut.conf.d/` firmware override is therefore expected for the default VM graphics configuration. Unlike physical Intel or AMD graphics, the guest normally does not need to include host GPU firmware in its initramfs.

If the VM is configured with a different virtual GPU or GPU passthrough, however, its firmware requirements may change. In particular, **VFIO/GPU passthrough should be treated as a separate hardware profile**, rather than adding physical-GPU firmware to the generic VM configuration.

---

## Building

A complete kernel build is made by running:

```sh
export LOCALVERSION=
make <your-selected-defconfig>
make -j$(nproc)
make modules_install
```

The first command generates the vendor-specific Vind configuration, the second builds the kernel, and the third installs kernel modules into the target filesystem (only relevant if `CONFIG_MODULES=y`; the minimal configurations are largely monolithic, so this step may be a no-op).

## Installing the Kernel Image

After a successful build, the kernel image is available at:

```text
arch/x86/boot/bzImage
```

Copy it to your `/boot` directory with a descriptive filename:

```sh
cp arch/x86/boot/bzImage /boot/vmlinuz-7.2.0-vind-intel-minimal
```

(Substitute `vind-amd-minimal` when building the AMD configuration.)

### Initramfs

The Vind minimal configurations expect to boot through an initramfs (`CONFIG_BLK_DEV_INITRD=y`), which mounts the root filesystem temporarily to run `fsck` before handing off control via `switch_root`. Without an initramfs, root-partition checks in `fstab` (`passno` > 0) will fail with a "device busy" error, since the kernel mounts root directly and exclusively.

Generate the initramfs with `dracut`:

```sh
dracut --force /boot/initramfs-7.2.0-vind-intel-minimal.img 7.2.0-vind-intel-minimal
```

Then regenerate the GRUB configuration so the new kernel/initramfs pair is picked up:

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

Confirm the resulting boot entry includes both a `linux` and an `initrd` line pointing to the new kernel and initramfs.

See the vendor-specific sections above ([Intel](#intel), [AMD](#amd)) for known firmware/initramfs caveats before considering a boot issue a kernel bug.

## Upstream

Vind tracks the mainline Linux kernel from:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

The `upstream` remote is used to follow changes from the mainline kernel.

To update the upstream references:

```sh
git fetch upstream
```

Vind-specific changes are maintained on the `vind` branch.

## Philosophy

Vind aims to keep its kernel configuration and modifications minimal.

Whenever possible, functionality is provided by the upstream Linux kernel rather than through unnecessary patches maintained by Vind.

The goal is a kernel that is:

* Minimal
* Predictable
* Suitable for Vind Linux
* Closely aligned with upstream Linux

## License

The Linux kernel is licensed under the GNU General Public License version 2, as described in `COPYING`.

Vind-specific changes remain subject to the licenses applicable to the files they modify.
