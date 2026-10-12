# Evidence and remaining limits

Tests were performed on October 11–12, 2026 on a Mid-2015 15-inch Retina
MacBookPro11,5, Omarchy 4.0.4-1, kernel 7.2.5-3-omarchy, encrypted Btrfs root.

## Storage/IOMMU

Before `intel_iommu=sp_off`, the internal AHCI SSD controller generated
DMAR fault reason `0x0c` (nonzero reserved PTE fields) alongside Btrfs checksum
errors. Disabling IOMMU superpages retained translated DMA domains and stopped
the observed faults. Subsequent fio read-verification, Btrfs scrub and device
counter checks were clean. SMART extended testing and three Memtest86+ passes
also passed. These observations support a superpage-path problem but do not
identify a precise kernel or firmware defect.

## Intel display

Blocking AMD did not initially make Intel drive the panel: the firmware's AMD
GOP framebuffer was still shown through simpledrm. Earlier reports of
"Intel-only" sleep tests described the loaded drivers, not verified panel ownership.

After selecting Intel through EFI, i915 detected the panel but rejected its
2880×1800 mode with `CLOCK_HIGH`. The sink supported four lanes at 2.7 Gbit/s;
Port A's allocation bit was clear, constraining the available lane allocation.
Setting the Haswell four-lane bit before i915 initialization allowed native
output. Readback was `0x81 → 0x91` in EFI; Linux subsequently used four lanes,
24 bpp, 2880×1800 at about 60 Hz. The helper refuses enabled A/E ports and
unexpected GPU identity before making that change.

Selecting Intel left an invisible stale AMD simpledrm monitor. Suppressing
simpledrm and efifb removed it. The unlock prompt remained invisible because
i915 probing deferred until apple-gmux loaded after root unlock. Including and
early-loading `apple-gmux` and `i915` restored a visible text prompt; adding
`splash` restored the graphical Omarchy prompt.

## S3 and Wi-Fi

| Test | Observation |
|---|---|
| AMD enabled, `sp_off`, S3 | GPU DMA-read fault `0x06`, ring timeout, AMD resume failure |
| AMD enabled, `sp_off`, s2idle | UVD firmware validation/resume failure |
| Actual Intel panel, AMD blocked, `sp_off`, raw S3 script | Display restored; one test spent about 126 seconds asleep, manually woken |
| Promoted default, normal S3, MAC randomization enabled | Display restored; firmware-command timeouts and `-5` errors left Wi-Fi dead until reboot |
| Same default, MAC randomization disabled | Two normal S3 cycles reconnected Wi-Fi in about 3–4 seconds; no MAC changes or firmware-command timeouts |

The last two cycles had suspend-entry-to-exit intervals of 28 and 47 seconds.
The raw S3 script skipped NetworkManager's normal preparation and therefore did
not establish reliable ordinary Wi-Fi suspend. Disabling scanning randomization
and setting the connection MAC to `permanent` removed the observed MAC changes;
the two later cycles succeeded. This association is encouraging but does not
prove MAC randomization is the only cause of Broadcom's intermittent failures.

No IOMMU faults, GPU hangs or storage I/O errors appeared in those successful
normal cycles. Broadcom ARP-table error `-52` and SATA ACPI
`unexpected _GTF length (8)` warnings remained.

An earlier attempt to unload `brcmfmac_wcc` live produced a kernel oops in
`_cfg80211_unregister_wdev` with a poisoned pointer (`0xdead000000000122`).
No live unload was retried. The working configuration uses no driver-reload
sleep hook.

## Not established

- Overnight or multi-hour S3 reliability and repeated-cycle endurance.
- Battery drain, AMD power-off, external display routing, hibernation.
- Other MacBook models, GPU identities, gmux versions or kernels.
- A precise firmware/IOMMU root cause or a kernel regression bisection.
- That a short successful cycle guarantees the Wi-Fi problem is resolved.

The repository intentionally excludes full private system journals, core dumps,
device serial numbers, MAC addresses, Wi-Fi SSIDs and machine-specific root paths.
