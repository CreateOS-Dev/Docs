# Drivers, filesystems, and storage

## Driver organization

Drivers are grouped under `kernel/drivers/` by device family: audio, clock,
console, input, network, platform, storage, and USB. Public kernel declarations
are separated under `kernel/header/drivers/`. Driver source presence is not a
claim that a device family is complete or works on physical hardware.

## Storage and filesystems

The kernel has block-device and partition-related code, a VFS, CFS5, and FAT32
support. FAT32 and CFS5 are not interchangeable: mounting, write behavior,
partition selection, error recovery, and media removal must be validated for
each path. GPT parsing and storage-controller support do not by themselves
provide a complete installer or universal disk support.

Test with disposable disk images. Check partition bounds and filesystem
metadata before writing. Preserve the user's persistent `build/disk.img`
unless rebuilding it is explicitly intended.

## USB and audio

The USB host-controller code detects UHCI, EHCI, and xHCI controllers, but
device transfers currently exist only for xHCI. The xHCI storage path supports
one full-, high-, or SuperSpeed USB Mass Storage device per controller using
SCSI Bulk-Only Transport, 512-byte sectors, and READ/WRITE(10); it exposes the
disk and recognized MBR/GPT partitions through the block layer. QEMU validates
SuperSpeed enumeration and a disposable-image sector write/read/restore cycle.
Multiple storage devices per controller, UAS, hubs, arbitrary USB classes,
and fully tested recovery from transfer errors are not supported.

xHCI recognizes HID boot-protocol keyboard and mouse interfaces, configures
their interrupt-IN endpoint, and routes parsed reports into the kernel keyboard
and mouse queues. `make check-usb-hid-qemu` verifies keyboard and mouse
enumeration and interrupt endpoint setup in separate QEMU runs; actual input
delivery still needs end-to-end validation. The current xHCI device state
supports only one USB device slot per controller, so storage, keyboard, and
mouse cannot yet all be used simultaneously on the same controller. UHCI/EHCI
transfers, HID report descriptors outside boot protocol, composite/multi-
interface HID, hubs, and real-hardware behavior remain unsupported or
unvalidated. Audio currently contains early PCI/AC'97 probing and PC-speaker
tone support, not a complete audio stack, codec ecosystem, mixer, or playback
API.

## Hardware validation

Use QEMU to establish a repeatable baseline, then test each supported device
and failure mode on real hardware before making compatibility claims. Record
PCI vendor/device IDs, controller mode, media, and the exact test performed.
The guest `lspci` command lists PCI BDFs, vendor/device IDs, and class codes
without exposing BAR addresses to applications. Use this output to identify
the exact onboard Wi-Fi, Bluetooth, audio, graphics, and USB controllers on a
target PC before selecting or claiming driver support. This does not enumerate
I2C/HID-over-I2C devices or prove that any listed device is supported.

HPET is initialized as an optional 64-bit monotonic clock when the ACPI HPET
table provides a supported memory-mapped counter. The PIT remains active as
the interrupt and timeout source; unsupported HPET hardware falls back to PIT
time. Its MMIO pages are mapped uncached through the VMM. The `clock` command
reports monotonic elapsed milliseconds. HPET initialization and counter
behavior still require validation on physical hardware.

PCI memory BARs are mapped uncached through the VMM, including 64-bit BARs
above 4 GiB when their address fits the low canonical identity-mapped range.
The Terra Mobile 360-11V3 inventory supplied so far identifies Jasper Lake
xHCI `8086:4ede`, Serial IO I2C `8086:4ee8` (BAR at `0x600113f000`), audio
`8086:4ec8`, UHD graphics `8086:4e71`, and Wi-Fi 6 AX201 `8086:4f30`.
PCI identification is not equivalent to a working device driver: the generic
xHCI path is present, but the Jasper Lake I2C/HID, HDA, graphics, and AX201
drivers are not implemented or validated on this machine. I2C-HID discovery
also requires ACPI child-device resources and descriptors, which are not
provided by `lspci`.
