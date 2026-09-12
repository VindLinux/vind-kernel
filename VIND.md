# Vind Kernel

This repository contains the Linux kernel used by **Vind Linux**.

Vind maintains a small set of kernel-specific changes on top of the upstream Linux kernel, keeping the upstream source as the base whenever possible.

For build failures, boot issues, or anything that looks like a kernel bug, check **[VIND-TROUBLESHOOTING.md](VIND-TROUBLESHOOTING.md)** before assuming it's new — most gotchas hit here have already been diagnosed there.

## Version

- **Linux:** 7.2.0
- **Architecture:** x86_64

This number can drift out of sync with the actual checked-out source if this document isn't updated on every version bump. Before naming a built image or an initramfs, confirm the real version with `make kernelversion` inside the kernel tree rather than trusting this section — `modules_install` output (`/lib/modules/<version>/`) is also authoritative and worth cross-checking against.

## Configuration

Vind provides minimal x86_64 kernel configurations, split by CPU vendor, plus a hardware-agnostic Generic profile for distro-wide builds. See [Intel](#intel), [AMD](#amd), [VM](#vm), and [Generic](#generic) below for details specific to each.

Hardware-specific driver configuration beyond what each vendor baseline enables is the user's responsibility. Before building the kernel for a physical machine, review the hardware and enable any additional required drivers in `make menuconfig`, `make nconfig`, or another kernel configuration interface. Pay particular attention to:

- Storage controllers (e.g. `SATA_AHCI`, `BLK_DEV_NVME`).
- Wi-Fi support and device drivers (e.g. `WLAN`, `CFG80211`, `MAC80211`, and chipset drivers).
- Input (`SERIO_I8042` + `KEYBOARD_ATKBD` for internal laptop keyboards, `USB_SUPPORT` + `HID_SUPPORT` + `USB_HID` for USB peripherals).
- Graphics driver matching the actual GPU.

The minimal configurations should be treated as a baseline rather than a universal hardware configuration.

All four profiles below are monolithic by design: `CONFIG_MODULES` is unset and every driver they enable is builtin (`=y`). No profile currently produces a `.ko`, and `modules_install` is a no-op for all of them (see [VIND-TROUBLESHOOTING.md](VIND-TROUBLESHOOTING.md) for why this matters more than it sounds like it should).

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

Known firmware caveat for this profile: [`.zst`-compressed firmware missing from initramfs](VIND-TROUBLESHOOTING.md#zst-compressed-firmware-missing-from-initramfs-intelamd).

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

Known caveats for this profile: [`.zst`-compressed firmware](VIND-TROUBLESHOOTING.md#zst-compressed-firmware-missing-from-initramfs-intelamd), [`CONFIG_WERROR` vs amdgpu DC/DML](VIND-TROUBLESHOOTING.md#config_werror-vs-amdgpu-dcdml-stack-frame-size).

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

The VM profile does not require physical GPU firmware — `DRM_VIRTIO_GPU`/`DRM_BOCHS`/`DRM_CIRRUS_QEMU` need none of it. If the VM uses GPU passthrough instead of these, treat that as a separate hardware profile rather than extending this one.

---

## Generic

- **Status:** available
- **Config file:** `arch/x86/configs/vind_x86_64_generic_minimal_defconfig`
- **Generate with:** `make vind_x86_64_generic_minimal_defconfig`

### Coverage

A union of the Intel, AMD, and VM baselines, for distro-wide images that must boot on any of the three targets without a rebuild. Same monolithic approach as the other three profiles — everything below is builtin (`=y`):

- **All hardware from all three targets:** both GPU drivers (`DRM_AMDGPU`, `DRM_I915`, plus the VM ones `DRM_VIRTIO_GPU`/`DRM_BOCHS`/`DRM_CIRRUS_QEMU`), both NIC sets (`E1000E`/`IGC`/`R8169` and `VIRTIO_NET`/`E1000`), `IWLWIFI`, both storage controller families (`SATA_AHCI`, `ATA_PIIX`, `BLK_DEV_NVME`, `VIRTIO_BLK`), both platform I2C/sensor chips (`I2C_I801`, `I2C_PIIX4`, `SENSORS_K10TEMP`), sound (`SND_HDA_INTEL`), and USB host controllers (`USB_XHCI_HCD`, `USB_UHCI_HCD`).
- **CPU vendor support is unconditional:** unlike the vendor-specific profiles, nothing here disables `CPU_SUP_INTEL` or equivalent AMD paths — both `CONFIG_X86_INTEL_PSTATE` and `CONFIG_X86_AMD_PSTATE` are built in side by side (each CPU only loads its own driver at runtime), with `schedutil` as the default governor.
- **Virtualization:** guest-side paravirtualization (`HYPERVISOR_GUEST`, `PARAVIRT`, `PARAVIRT_SPINLOCKS`, `KVM_GUEST`, `PVH`) is enabled unconditionally, same as the VM profile — it's a no-op on bare metal. Generic additionally enables **KVM host** support (`CONFIG_KVM`, `CONFIG_KVM_INTEL`, `CONFIG_KVM_AMD`, `CONFIG_VHOST_NET`, `CONFIG_TUN`, `CONFIG_BRIDGE`), which none of the vendor-specific profiles provide.
- **VirtIO:** the full stack from the VM profile (`VIRTIO_MENU`, `VIRTIO_PCI`, `VIRTIO_BALLOON`, `VIRTIO_INPUT`, `VIRTIO_MMIO`, `HW_RANDOM` + `HW_RANDOM_VIRTIO`, `NET_9P`/`NET_9P_VIRTIO`/`9P_FS`) is included, so a Generic image also works as a VM guest without a separate build.
- `CONFIG_WERROR` is unset here for the same reason as AMD — see [`CONFIG_WERROR` vs amdgpu DC/DML](VIND-TROUBLESHOOTING.md#config_werror-vs-amdgpu-dcdml-stack-frame-size).

Known caveats for this profile: [`.zst`-compressed firmware](VIND-TROUBLESHOOTING.md#zst-compressed-firmware-missing-from-initramfs-intelamd), [`CONFIG_WERROR` vs amdgpu DC/DML](VIND-TROUBLESHOOTING.md#config_werror-vs-amdgpu-dcdml-stack-frame-size), and — specific to this profile — [GPU firmware on a monolithic build](VIND-TROUBLESHOOTING.md#gpu-firmware-on-a-monolithic-build-generic), a decision you need to make explicitly before shipping an image.

---

## Building

A complete kernel build is made by running:

```sh
export LOCALVERSION=
make <your-selected-defconfig>
make -j$(nproc)
make modules_install
```

The first command generates the vendor-specific Vind configuration; the second builds the kernel itself; the third installs kernel modules into the target filesystem. Since every profile is monolithic (`CONFIG_MODULES` unset), this step is a no-op today for all four — see [VIND-TROUBLESHOOTING.md](VIND-TROUBLESHOOTING.md#config_modules-silently-unset-m-gets-promoted-to-y) if you're trying to reintroduce modules and it isn't behaving as expected.

## Installing the Kernel Image

After a successful build, the kernel image is available at:

```text
arch/x86/boot/bzImage
```

Copy it to your `/boot` directory with a descriptive filename:

```sh
cp arch/x86/boot/bzImage /boot/vmlinuz-7.2.5-vind-intel-minimal
```

(Swap `vind-intel-minimal` for whichever defconfig you actually built — `vind-amd-minimal`, `vind-vm-minimal`, or `vind-generic-minimal`. Confirm the version number matches `make kernelversion` output, not just what this doc says — see [VIND-TROUBLESHOOTING.md](VIND-TROUBLESHOOTING.md#version-string-mismatch-between-vindmd-and-the-actual-source-tree).)

### Initramfs (optional)

An initramfs is **not required** to boot a Vind kernel on any of the four profiles. `CONFIG_BLK_DEV_INITRD=y` is enabled so the option is there, but every profile is monolithic on purpose, and a kernel with everything it needs to find and mount root already built in can hand off straight from GRUB with no initramfs stage at all.

Reach for one when something has to run in userspace *before* root can be mounted — most commonly:

- An encrypted or network-backed root (`dm-crypt`, LVM, NFS root, etc.).
- GPU firmware staging on the [Generic](#generic) profile — see [VIND-TROUBLESHOOTING.md](VIND-TROUBLESHOOTING.md#gpu-firmware-on-a-monolithic-build-generic).
- Anything else that needs to happen ahead of `switch_root`.

If none of that applies to your build, skip this section entirely and go straight to regenerating GRUB below. If it does, generate the initramfs with `dracut`:

```sh
dracut --force /boot/initramfs-7.2.5-vind-intel-minimal.img 7.2.5-vind-intel-minimal
```

And if you are using musl libc generate the initramfs with DRACUT_LDCONFIG=true:

```sh
DRACUT_LDCONFIG=true dracut --force /boot/initramfs-intel.img 7.2.5-vind-intel-minimal
```

(Same substitution as above — match the image and version string to the kernel you just built, and to what `/lib/modules/` actually contains after `modules_install`.)

Either way — with or without an initramfs — regenerate the GRUB configuration so it picks up the new kernel:

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

If you built an initramfs, confirm the resulting boot entry has both a `linux` and an `initrd` line pointing at the new kernel and initramfs. If you skipped it, the entry should have only the `linux` line — no `initrd` line is expected or needed.

## Upstream

Vind tracks the mainline Linux kernel from:

```text
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

This is followed through an `upstream` remote, kept up to date with:

```sh
git fetch upstream
```

Vind-specific changes live entirely on the `vind` branch, on top of whatever upstream commit that remote currently points to.

## Philosophy

Vind aims to keep its kernel configuration and modifications minimal, leaning on what upstream Linux already provides rather than maintaining patches for functionality that already exists there.

The goal is a kernel that is:

* Minimal
* Predictable
* Suitable for Vind Linux
* Closely aligned with upstream Linux

## License

The Linux kernel is licensed under the GNU General Public License version 2, as described in `COPYING`.

Vind-specific changes remain subject to the licenses applicable to the files they modify.
