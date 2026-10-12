# Intel display and S3 on MacBookPro11,5

An experimental EFI helper and documented Omarchy configuration for the
**MacBook Pro (Retina, 15-inch, Mid 2015), model `MacBookPro11,5`**.
The helper selects Intel before Linux starts. On the tested machine, the full
configuration restored native Intel display output, graphical LUKS unlock,
and normal S3/deep suspend with Wi-Fi reconnection.

**Only MacBookPro11,5 is supported and tested.** The helper refuses other DMI
models, an unexpected gmux version, a different Intel GPU, or enabled display
ports. It is not a general tool for all Intel Macs, T2 Macs or Apple Silicon.
The AMD driver remains disabled; the helper does not power the AMD GPU off.
External displays and overnight sleep have not been validated.

## The working combination

| Component | Configuration on the tested machine |
|---|---|
| Firmware display selection | This EFI helper selects Intel DDC and internal panel |
| Haswell display lanes | Port A gets four lanes before i915 initializes |
| Early display drivers | `MODULES+=(apple-gmux i915)` in mkinitcpio |
| AMD drivers | `amdgpu` and `radeon` blocked |
| Intel IOMMU | `intel_iommu=sp_off` retains translation, disables superpages |
| Firmware framebuffer | Suppress `simpledrm_platform_driver_init` and `efifb_driver_init` |
| Graphical unlock | Retain `splash`; tested with synchronous `initramfs_async=0` |
| Suspend | `mem_sleep_default=deep` |
| Broadcom Wi-Fi | Disable scanning MAC randomization and use permanent connection MAC |

Each part addressed a different observed failure. This does not establish a
single root cause for the storage, AMD resume, display and wireless problems.
See [results](docs/results.md) and [installation/recovery](docs/setup.md).

## What the helper does

1. Checks SMBIOS product `MacBookPro11,5`.
2. Loads `\EFI\Linux\omarchy_linux-omarchy.efi` from the same EFI partition
   and forwards the Limine command line to it.
3. Calls Apple's Set OS protocol with `Mac OS X 10.9` and `Apple Inc.`.
4. Checks indexed gmux version 4.0.20 at I/O base `0x700`.
5. Verifies Intel `8086:0d26` at `0000:00:02.0` through EFI PCI I/O. Checks
   that Haswell Port A and E are disabled and readable; sets only Port A's
   four-lane allocation bit, then verifies readback.
6. Selects Intel DDC (`1`) and internal display (`2`) through gmux and verifies
   them, then starts the Linux image.

Checkpoints are appended and flushed to
`\EFI\Omarchy\intel-before-linux.log` before Linux starts. That directory
must exist for logging. The helper attempts to restore lane and mux settings
on failure or an unexpected return. It does not write NVRAM, change GPU power,
switch the external-display mux, or alter the disk encryption configuration.

Apple's GOP may still describe the AMD framebuffer after the mux changes.
The Linux framebuffer suppression prevents that stale display from appearing
as an extra monitor. Early `apple-gmux` loading is essential: on this machine
i915 deferred probing until gmux appeared, leaving the LUKS prompt invisible
when gmux was only available after unlocking root.

## Build

Requires x86-64 Linux, GCC, GNU binutils and GNU-EFI. On Omarchy:

```sh
omarchy pkg add gnu-efi
./build.sh
```

The output is `intel-before-linux.efi`. No prebuilt EFI executable is shipped.
The build uses GNU-EFI's installed x86-64 headers, startup and libraries; it
checks for unresolved symbols and prints the output hash and file type.
The helper is unsigned; the tested machine has Secure Boot disabled.

Keep a working boot entry that bypasses the helper before testing. Build and
install a separate test entry first, following [the setup guide](docs/setup.md).
The helper's UKI path is fixed in source; other bootloaders, kernels or paths
require adapting it. Do not simply copy these settings to a different model.

## Tested environment and limits

- Omarchy `4.0.4-1`, `7.2.5-3-omarchy`, Limine, encrypted Btrfs root.
- Intel Core i7-4980HQ, Iris Pro 5200 (`8086:0d26`), AMD Radeon R9 M370X
  (`1002:6821`), Broadcom BCM43602, Apple PCIe AHCI SSD.
- Native Intel panel: 2880×1800 at approximately 60 Hz, four lanes at 2.7 Gbit/s.
- Graphical unlock and default-entry boot verified on October 11–12, 2026.
- Two normal S3 cycles after disabling MAC randomization reconnected Wi-Fi
  approximately 3–4 seconds after resume. Entry-to-exit intervals were 28 and
  47 seconds. A previous raw-kernel test spent about 126 seconds asleep but
  bypassed NetworkManager's normal preparation and is weaker Wi-Fi evidence.
- These are a few successful tests, not proof of long-term reliability.
  Broadcom ARP-table warnings and SATA ACPI `_GTF` warnings still occurred.

Dedicated display/S3 report: [Omarchy #15103](https://github.com/omacom/omarchy/issues/15103).

Related Omarchy reports: [IOMMU/storage #14160](https://github.com/omacom/omarchy/issues/14160),
[AMD boot #12550](https://github.com/omacom/omarchy/issues/12550),
[early KMS #13510](https://github.com/omacom/omarchy/issues/13510),
[Broadcom suspend #11377](https://github.com/omacom/omarchy/issues/11377),
and [AMD power tradeoff #11382](https://github.com/omacom/omarchy/issues/11382).

GPL-2.0-only; see [LICENSE](LICENSE), [ATTRIBUTION.md](ATTRIBUTION.md) and
the third-party notices in `LICENSES/`.
