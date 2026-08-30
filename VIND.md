# Vind Kernel

This repository contains the Linux kernel used by **Vind Linux**.

Vind maintains a small set of kernel-specific changes on top of the upstream Linux kernel, keeping the upstream source as the base whenever possible.

## Version

- **Linux:** 7.2.0
- **Architecture:** x86_64

## Configuration

Vind provides minimal x86_64 kernel configurations, split by CPU vendor:

```sh
make vind_x86_64_intel_minimal_defconfig
```

An AMD equivalent (`vind_x86_64_amd_minimal_defconfig`) is planned but not yet available.

The Intel configuration is located at:

```text
arch/x86/configs/vind_x86_64_intel_minimal_defconfig
```

This configuration is intended as a starting point for Vind Linux systems running on Intel hardware.

### Hardware Support

The Vind configuration is deliberately minimal and does **not** attempt to enable every available Linux driver.

**Hardware-specific driver configuration is the user's responsibility.**

Before building the kernel for a physical machine, review the hardware and enable the required drivers in `make menuconfig`, `make nconfig`, or another kernel configuration interface. Pay particular attention to:

- Storage controllers (e.g. `SATA_AHCI`, `BLK_DEV_NVME`) — including vendor-specific bridging such as Intel VMD (`CONFIG_VMD`), which hides NVMe controllers on some laptops even without RAID configured.
- Input (`SERIO_I8042` + `KEYBOARD_ATKBD` for internal laptop keyboards, `USB_SUPPORT` + `HID_SUPPORT` + `USB_HID` for USB peripherals).
- Graphics driver matching the actual GPU (e.g. `DRM_I915` for Intel).

For example:

```sh
make menuconfig
```

The minimal configuration should therefore be treated as a baseline rather than a universal hardware configuration.

## Building

A complete kernel build is made by running:

```sh
make vind_x86_64_intel_minimal_defconfig
make -j$(nproc)
make modules_install
```

The first command generates the Vind Intel minimal configuration, the second builds the kernel, and the third installs kernel modules into the target filesystem (only relevant if `CONFIG_MODULES=y`; the minimal configuration is largely monolithic, so this step may be a no-op).

## Installing the Kernel Image

After a successful build, the kernel image is available at:

```text
arch/x86/boot/bzImage
```

Copy it to your `/boot` directory with a descriptive filename:

```sh
cp arch/x86/boot/bzImage /boot/vmlinuz-7.2.0-intel-minimal
```

### Initramfs

The Vind minimal configuration expects to boot through an initramfs (`CONFIG_BLK_DEV_INITRD=y`), which mounts the root filesystem temporarily to run `fsck` before handing off control via `switch_root`. Without an initramfs, root-partition checks in `fstab` (`passno` > 0) will fail with a "device busy" error, since the kernel mounts root directly and exclusively.

Generate the initramfs with `dracut`:

```sh
dracut --force /boot/initramfs-7.2.0-vind.img 7.2.0-intel-minimal
```

Then regenerate the GRUB configuration so the new kernel/initramfs pair is picked up:

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

Confirm the resulting boot entry includes both a `linux` and an `initrd` line pointing to the new kernel and initramfs.

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
