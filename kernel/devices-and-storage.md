# Devices and storage

Device support is still developing. QEMU is the standard test machine, and a
successful QEMU run does not guarantee support for a similarly named physical
device.

## Device discovery

PCI enumeration is handled by the platform code under `kernel/drivers/` and
`kernel/header/drivers/`. Drivers attach to devices they recognize and
register services such as network interfaces or block storage. `lspci` shows
the devices the guest found.

The source tree includes console/framebuffer, keyboard, mouse, clock, ACPI,
PCI, USB, audio, storage, and network code. Individual functions may be
limited or polling-based; check the relevant driver before depending on a
specific device or interrupt path.

## Disk controllers

The kernel has ATA, AHCI, and NVMe-related storage code. CFS5 uses the block
device selected by the current boot setup. QEMU's IDE disk is the usual
development configuration. Physical controller discovery, firmware quirks,
and hot-plug behavior need hardware-specific testing.

## CFS5 root filesystem

The root filesystem is a CFS5 image at `build/disk.img`. The kernel mounts it
at `/`. The on-disk format uses 4096-byte blocks, an inode table, allocation
bitmaps, and extents. Host-side image tools in `rootfs_maker/tools/` create,
populate, inspect, and check the image.

Useful commands:

```sh
make disk-image
build/rootfs_maker/host/cfs5tool info build/disk.img
build/rootfs_maker/host/cfs5tool check build/disk.img
```

The exact host tool path depends on the build output. `make host-tools` builds
the tools used by the rootfs maker. Keep a backup of a disk image that
contains files you care about; recreating it discards that data.

## USB

USB and HID support is under `kernel/drivers/usb/`. Keyboard and mouse
emulation in QEMU are useful for development, but device enumeration,
controller variants, hubs, and physical USB storage need separate testing.

## Fonts and cursor assets

Font source and embedded font data are in `kernel/header/fonts/`; cursor and
keyboard assets for the guest are staged by `rootfs_maker/`. User-facing
pointer and keyboard preferences are stored in the root filesystem. When an
asset is changed, rebuild or refresh the staged rootfs so the guest sees it.
