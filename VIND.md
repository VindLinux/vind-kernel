# Vind Kernel

This repository contains the Linux kernel used by **Vind Linux**.

Vind maintains a small set of kernel-specific changes on top of the upstream Linux kernel, keeping the upstream source as the base whenever possible.

## Version

- **Linux:** 7.2.0
- **Architecture:** x86_64

## Configuration

Vind provides a minimal x86_64 kernel configuration:

```sh
make vind_x86_64_minimal_defconfig
```

The configuration is located at:

```text
arch/x86/configs/vind_x86_64_minimal_defconfig
```

This configuration is intended as a starting point for Vind Linux systems.

### Hardware Support

The Vind configuration is deliberately minimal and does **not** attempt to enable every available Linux driver.

**Hardware-specific driver configuration is the user's responsibility.**

Before building the kernel for a physical machine, review the hardware and enable the required drivers in `make menuconfig`, `make nconfig`, or another kernel configuration interface.

For example:

```sh
make menuconfig
```

The minimal configuration should therefore be treated as a baseline rather than a universal hardware configuration.

## Building

A complete kernel building is made by running:

```sh
make vind_x86_64_minimal_defconfig
make -j$(nproc)
make modules_install
```

The first command generates the Vind minimal x86_64 configuration, the second builds the kernel, and the third installs the kernel modules into the target filesystem.

Additional build and installation steps depend on the target Vind Linux system.

## Installing the Kernel Image

After a successful build, the kernel image is available at:

```text
arch/x86/boot/bzImage
```

Copy it to your /boot directory with a descriptive filename:

```sh
cp arch/x86/boot/bzImage /boot/vmlinuz-7.2.0-vind
```

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

