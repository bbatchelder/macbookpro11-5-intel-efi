# Installation and recovery

This is the procedure used for this machine, not a universal installer.
Understand your existing Limine and mkinitcpio configuration before editing it.
The helper deliberately supports only `MacBookPro11,5` and the checked devices.

## Preserve a boot that bypasses the helper

1. Save `/boot/limine.conf`, your current UKI and any configuration files you
   will change. Keep the old UKI at a separate path; an entry that points at the
   usual UKI path is not a preserved image once that path is rebuilt.
2. Keep a recovery entry with the old image and its original command line.
   Set its `path:` verification hash to that image's `b2sum` result.
3. Ensure you can select recovery in Limine. Do not change the default yet.

The working installation retained a direct-UKI recovery entry that bypasses
all EFI-helper display writes. No NVRAM state needs clearing on recovery.

## Early drivers and boot parameters

Copy `config/zz-intel-display.conf` into `/etc/mkinitcpio.conf.d/`. The array
addition includes and loads `apple-gmux` before `i915` and before the boot hooks.
Rebuild the actual UKI used by the helper, using your installed Limine/mkinitcpio
kernel-update tooling. `mkinitcpio -P` does nothing if the machine has no presets;
check what the installed pacman hooks use rather than assuming a preset exists.

The tested setup blocked AMD both through `config/disable-amd-gpu.conf` in
`/etc/modprobe.d/` and through the command line. Its relevant command-line
additions were:

```text
modprobe.blacklist=amdgpu,radeon module_blacklist=amdgpu,radeon intel_iommu=sp_off initcall_blacklist=simpledrm_platform_driver_init,efifb_driver_init mem_sleep_default=deep initramfs_async=0 splash
```

Keep your own encryption, root filesystem, swap/resume and other essential
parameters. Remove conflicting `intel_iommu=off` or `mem_sleep_default=s2idle`
settings if you intend this S3 test. Do not use `nomodeset`: i915 must drive the
panel. The named initcalls were verified on `7.2.5-3-omarchy`; they may change
with another kernel, so verify their names in `/proc/kallsyms` before reusing.

The command-line configuration example in `config/intel-display.conf` supplies
only the hardware additions; retain your existing Omarchy splash and synchronous
initramfs settings. Avoid duplicating these settings in multiple drop-ins.

## Build and test the helper

Build with `./build.sh`. Create `/boot/EFI/Omarchy/` and install the output there
as `intel-before-linux-default.efi`. The Linux image must be at
`/boot/EFI/Linux/omarchy_linux-omarchy.efi` because that path is fixed in source.

Add a **separate** test entry in `/boot/limine.conf`:

```text
/Omarchy (Intel EFI test)
protocol: efi
path: boot():/EFI/Omarchy/intel-before-linux-default.efi#REPLACE_WITH_BLAKE2B_HASH
cmdline: YOUR_EXISTING_ROOT_AND_ENCRYPTION_ARGUMENTS PLUS_THE_PARAMETERS_ABOVE
```

Those placeholders must be replaced. Generate the helper hash with
`b2sum /boot/EFI/Omarchy/intel-before-linux-default.efi`. Refresh Limine's config
enrollment using your installed `limine-enroll-config` command. Use the same
boot-partition lock as the installed tooling when changing boot files, and do
not run a separate lock-taking command from inside an existing lock.

Reboot into the test entry. A brief blank screen during firmware-to-i915 handoff
is expected; a missing unlock prompt is not a successful test. Use the preserved
recovery entry if needed. Check the EFI checkpoint log and the Linux journal.

## Disable Wi-Fi MAC randomization

Install `config/90-wifi-permanent-mac.conf` in `/etc/NetworkManager/conf.d/`.
Global defaults do not override an explicit connection-profile setting, so set
the saved Wi-Fi profile too:

```sh
nmcli connection modify "YOUR_WIFI_PROFILE" 802-11-wireless.cloned-mac-address permanent
nmcli general reload conf
NetworkManager --print-config
```

These commands may require authorization. Verify both
`wifi.scan-rand-mac-address=no` and `wifi.cloned-mac-address=permanent` in the
effective configuration and `permanent` in the profile. This exposes the
hardware MAC during scanning and connecting; the deliberate tradeoff here is
avoiding MAC changes while investigating this Broadcom resume failure.

Wi-Fi power saving was already disabled on the tested installation, and it
retained Omarchy's existing `brcmfmac feature_disable=0x82000`. Those settings
were not varied in the successful MAC-randomization comparison.
Do not unload `brcmfmac` or `brcmfmac_wcc` live to imitate a reset: an earlier
unload on this kernel oopsed in `_cfg80211_unregister_wdev`.

## Verify and promote only after testing

Verify that the connected, enabled internal eDP connector belongs to i915 and
that Hyprland uses it. Intel render nodes alone do not establish display
ownership. On the tested installation:

```sh
hyprctl monitors
cat /sys/power/mem_sleep
journalctl -b -k | rg 'Initialized i915|apple_gmux|i915drmfb|PM: suspend'
nmcli device status
```

`[deep]` should be selected for S3. Test an ordinary lid-close/open or
`systemctl suspend` cycle, including Wi-Fi connectivity after resume. Raw writes
to `/sys/power/state` bypass normal NetworkManager/systemd preparation and are
not equivalent Wi-Fi validation. Check for DMAR, GPU and firmware-command
timeouts after each cycle.

For update persistence, place a root-owned copy of the built helper at
`/usr/local/share/omarchy-intel-display/intel-before-linux-default.efi`. The
provided `hooks/88-omarchy-intel-display` can be installed executable in
`/etc/boot/hooks/post.d/`. It uses the existing Limine boot lock and runs before
config enrollment; it redirects **only** `Omarchy/linux-omarchy` through the
helper, then continues to use the maintained UKI at its normal path. Inspect its
hardcoded entry/path assumptions before installing. It copies the helper from
the root-owned source on every invocation, so the source must remain available.

Keep early-driver and command-line drop-ins persistent. Regenerate the UKI/entry
with the installed updater and verify the helper hash, default selection, and
recovery entries. The tested machine's `--add-uki` regeneration preserved the
helper entry, but future kernel changes still require reboot validation.

## Rollback

Disable or remove the `88-omarchy-intel-display` boot hook before regenerating
entries. Restore the backed-up Limine/mkinitcpio/modprobe settings and UKI, or
select the preserved direct-UKI recovery entry. Restore your previous NetworkManager
profile MAC setting and remove the new MAC drop-in if desired, then reload its
configuration. Retain the recovery image until the restored default is tested.
